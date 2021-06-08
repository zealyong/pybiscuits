<!--
 * @Description: 
 * @version: 
 * @Author: GongZiyao
 * @Date: 2021-06-07 10:10:36
 * @LastEditors: GongZiyao
 * @LastEditTime: 2021-06-08 16:12:43
-->

# django 启动和路由寻址机制

## 1. 启动运行命令发生了什么？
```python
python manage.py runserver

# manage.py
# 入口文件中实际执行了该方法
# 此处argv为['manage.py', 'runserver']
execute_from_command_line(sys.argv)

# core.management.__init__.py
def execute_from_command_line(argv=None):
    utility = ManagementUtility(argv)
    utility.execute()


class ManagementUtility:

 def execute(self):

    # 指定py路径
    # 加载全局变量 settings.INSTALLED_APPS
    ''' 省略部分代码 ''' 

    # 执行启动，这里只截取了部分核心代码
    subcommand = self.argv[1]
    if subcommand == 'runserver:
        django.setup()  
        # django.setup 主要功能为创建 settings.INSTALLED_APPS 中配置的app对象以供实际业务调用


```


## 2. 路由寻址和视图函数的执行

![Image text](../pictures/django_request_process.png)

下面上源码：
```python
# core.handlers.base

# 处理请求核心代码的基类
class BaseHandler:

    # 刚方法一般在网关接口文件中执行
    def load_middleware(self):
        #  Must be called after the environment is fixed (see __call__ in subclasses).
        #  官方注释指明了子类当中的 __call__方法包含实际业务处理过程，比如process_request, process_response

        # 核心私有函数 get_response，本文以该方法为例，与源码略有不同
        get_response = self._get_response()
        handler = get_response

        # 装载 settings.MIDDLEWARE 中配置的中间件到私有属性  _middleware_chain
        for middleware_path in reversed(settings.MIDDLEWARE):

            # 这里主要是对配置的中间件地址进行一些列处理，返回生产实例的参数，代码暂省略
            adapted_handler = Func(middleware_path)
            """ 省略部分代码"""

            # 工厂模式返回的中间件实例,，取出对应的方法供给后续执行
            mw_instance = middleware(adapted_handler)
            if hasattr(mw_instance, 'process_view'):
                self._view_middleware.insert(mw_instance)
            if hasattr(mw_instance, 'process_template_response'):
                self._template_response_middleware.append(
                    self.adapt_method_mode(is_async, mw_instance.process_template_response),
                )
            if hasattr(mw_instance, 'process_exception'):
                self._exception_middleware.append(
                    self.adapt_method_mode(False, mw_instance.process_exception),
                )

        ''' 省略部分代码 ''' 
        
        self._middleware_chain = handler


    # 映射路由的逻辑与业务处理最核心的方法
    def _get_response(self, request):

        response = None

        # 路由寻址找到对应view
        callback, callback_args, callback_kwargs = self.resolve_request(request)

        # 请求过程中依次执行配置的中间件，view方法
        for middleware_method in self._view_middleware:
            response = middleware_method(request, callback, callback_args, callback_kwargs)

            # 有任意一个中间件生成了response，则后续中间件全部不执行直接返回
            if response:
                break

        # 该行主要是截获视图返回None的异常，若因上述中间件处理返回静态response则直接return
        # 否则在下一行开始处理执行的流程
        self.check_response(response, callback)


        # 依次执行渲染模板的中间件
        if hasattr(response, 'render') and callable(response.render):
            for middleware_method in self._template_response_middleware:
                # 中间件执行方法依次应用到response中去
                response = middleware_method(request, response)
                
            try:
                # 最终渲染模板
                response = response.render()
            except Exception as e:
                response = self.process_exception_by_middleware(e, request)
                if response is None:
                    raise

        return response

    def resolve_request(self, request):
        # url匹配处理函数

        if hasattr(request, 'urlconf'):
            urlconf = request.urlconf
            set_urlconf(urlconf)
            resolver = get_resolver(urlconf)
        else:
            resolver = get_resolver()
        resolver_match = resolver.resolve(request.path_info)
        request.resolver_match = resolver_match
        return resolver_match
```
```python
# urls.resolvers.resolve

class URLResolver:

    # 进行实际URL匹配的方法
    
    ''' 省略部分代码 ''' 

    def resolve(self, path):
        path = str(path)  # path may be a reverse_lazy object
        tried = []
        match = self.pattern.match(path)
        if match:
            new_path, args, kwargs = match

            # 遍历 self.url_patterns 进行匹配，返回合适的ResolverMatch对象
            # 虽然遍历不太优雅，不过django使用了lru缓存进行提速
            for pattern in self.url_patterns:
                try:
                    sub_match = pattern.resolve(new_path)
                except Resolver404 as e:
                    self._extend_tried(tried, pattern, e.args[0].get('tried'))
                else:
                    if sub_match:
                        # 该对象包含执行方法，入参，url等属性
                        return ResolverMatch(...)
```


整个调用流程有个类处理函数描述的很清晰
中间件与视图函数的处理顺序为：
process_request() -> process_view() -> **view_func() -> process_template_response() -> process_response()
若执行过程中出现系统异常或者任意一个中间件返回结果不为空，则直接调用返回


```python
# utils.decorators.py

def make_middleware_decorator(middleware_class):
    def _make_decorator(*m_args, **m_kwargs):
        def _decorator(view_func):
            middleware = middleware_class(view_func, *m_args, **m_kwargs)

            @wraps(view_func)
            def _wrapped_view(request, *args, **kwargs):
                
                # steo1
                if hasattr(middleware, 'process_request'):
                    result = middleware.process_request(request)
                    if result is not None:
                        return result
                if hasattr(middleware, 'process_view'):
                    result = middleware.process_view(request, view_func, args, kwargs)
                    if result is not None:
                        return result
                try:
                    response = view_func(request, *args, **kwargs)
                except Exception as e:
                    if hasattr(middleware, 'process_exception'):
                        result = middleware.process_exception(request, e)
                        if result is not None:
                            return result
                    raise
                if hasattr(response, 'render') and callable(response.render):
                    if hasattr(middleware, 'process_template_response'):
                        response = middleware.process_template_response(request, response)
                    # Defer running of process_response until after the template
                    # has been rendered:
                    if hasattr(middleware, 'process_response'):
                        def callback(response):
                            return middleware.process_response(request, response)
                        response.add_post_render_callback(callback)
                else:
                    if hasattr(middleware, 'process_response'):
                        return middleware.process_response(request, response)
                return response
            return _wrapped_view
        return _decorator
    return _make_decorator

```