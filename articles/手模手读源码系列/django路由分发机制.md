<!--
 * @Description: 
 * @version: 
 * @Author: GongZiyao
 * @Date: 2021-06-07 10:10:36
 * @LastEditors: GongZiyao
 * @LastEditTime: 2021-06-08 17:20:53
-->

# django 路由寻址与处理流程

与网上大部分搜到的内容不同，本文主要以精简源码为主，介绍一下django处理请求的流程，中间件的细节先挖个坑~ 后续填上

![Image text](https://raw.githubusercontent.com/zealyong/pybiscuits/main/pictures/django_request_process.png)

这个图是网上流传比较广的图，写的也很清楚，简单介绍一下

1. 首先wsgi网关入口文件收到浏览器的请求，开始执行中间件中的process_request() 方法，执行顺序和中间件配置中的顺序保持一致（源码中直接for循环遍历）
2. 接下来执行之后依次执行各个中间件中的 process_view()
3. 执行实际的view方法，与model交互获取数据
4. 执行 process_template_response() 方法
5. 执行 process_response() 方法

需要注意的是：
1.上述步骤2,3,4过程中出现执行异常会直接跳到步骤5进行返回消息中间件的处理
2.步骤2,3,4过程中出现有返回值的情况也会直接跳到步骤5


下面上源码：
```python
# core.handlers.base

# 处理请求核心代码的基类
class BaseHandler:

    # 刚方法一般在网关接口文件中执行（上图中的wsgi）
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



整个调用流程有个类处理函数描述的很清晰（这里的源码不是每次请求都会走到，只是流程写的很清晰所以复制过来展示一下）

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
                
                # step1
                if hasattr(middleware, 'process_request'):
                    result = middleware.process_request(request)
                    if result is not None:
                        return result
                # step2
                if hasattr(middleware, 'process_view'):
                    result = middleware.process_view(request, view_func, args, kwargs)
                    if result is not None:
                        return result
                try:
                    # step3
                    response = view_func(request, *args, **kwargs)
                except Exception as e:
                    if hasattr(middleware, 'process_exception'):
                        result = middleware.process_exception(request, e)
                        if result is not None:
                            return result
                    raise
                if hasattr(response, 'render') and callable(response.render):
                    if hasattr(middleware, 'process_template_response'):
                        # step4
                        response = middleware.process_template_response(request, response)
                    if hasattr(middleware, 'process_response'):
                        def callback(response):
                            # step5
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