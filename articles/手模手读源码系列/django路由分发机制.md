<!--
 * @Description: 
 * @version: 
 * @Author: GongZiyao
 * @Date: 2021-06-07 10:10:36
 * @LastEditors: GongZiyao
 * @LastEditTime: 2021-06-07 19:19:06
-->

# django 启动和路由寻址机制

## 1. 启动运行命令发生了什么？
```
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


## 2. 路由寻址

```
# core.handlers.base

class BaseHandler:

    # 处理请求核心代码的基类

    def load_middleware():
        # 装载 settings.MIDDLEWARE 中配置的中间件到私有属性 _middleware_chain
        ''' 省略部分代码 ''' 

    def get_response(self, request):

        # 设置项目路由配置文件的存放地址，一般为url.py
        set_urlconf(settings.ROOT_URLCONF)

        # 依次执行配置的中间件，中间件可供执行前，返回时或异常时处理
        # 和本章主题无关 ，先埋个坑~
        response = self._middleware_chain(request) 
        response._resource_closers.append(request.close)

        # 判断业务执行返回
        if response.status_code >= 400:
            log_response(
                '%s: %s', response.reason_phrase, request.path,
                response=response,
                request=request,
            )
        return response

```