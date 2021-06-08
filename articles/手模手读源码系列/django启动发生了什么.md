<!--
 * @Description: 
 * @version: 
 * @Author: GongZiyao
 * @Date: 2021-06-08 16:59:52
 * @LastEditors: GongZiyao
 * @LastEditTime: 2021-06-08 17:00:00
-->

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