# python_log模块使用

```
import loguru

from loguru import logger

# 删除以前添加的处理程序
logger.remove(handler_id=None)
# 将日志输出到文件
trace = logger.add("E:/PythonCode/MOC/log_2021-3-28.log", rotation="500MB", encoding="utf-8", enqueue=True,
                   retention="10 days", level='ERROR')
# 日志信息格式化
logger.info('If you are using Python {version}, prefer {feature} of course!', version=platform.python_version(), feature='f-strings')

logger.info('This is info information')
logger.debug('This is debug information')
logger.info('This is info information')
logger.warning('This is warn information')
logger.error('This is error information')

# 异常追溯 , 记录报错信息
@logger.catch
def index_error(custom_list: list):
    for index in range(len(custom_list)):
        try:
            index_value = custom_list[index]
        except IndexError as e:
            logger.exception(e)
            break
        if custom_list[index] < 2:
            custom_list.remove(index_value)

        print(index_value)

trace = logger.add('2021-3-28.log',
                   format="{time:YYYY-MM-DD HH:mm:ss} {extra[ip]}  {extra[username]} {level} From {module}.{function} : {message}")

extra_logger = logger.bind(ip="192.168.0.1", username=" 张三 ")
extra_logger.info('This is info information')
extra_logger.bind(username=" 李四 ").error("This is error information")

extra_logger.warning('This is warn information')

# 停止日志记录到文件
logger.remove(trace)

```

