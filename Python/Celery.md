Celery 是一个分布式异步消息队列，通过它可以轻松的实现任务的异步处理
Celery可以使用RabbitMQ或者Redis作为broker消息转发器，以及backend结果记录
例如项目目录为：
```txt
|- celery_tasks
	|- __init__.py
	|- celery.py
	|- task01.py
	|- task02.py
|- check_result.py
|- produce_task.py
```

```python title:celery_tasks/celery.py
from celery import Celery  
  
cel = Celery('tasks',  
             broker='redis://localhost:6379/1',  
             backend='redis://localhost:6379/2',  
             include=[  
                 'celery_tasks.task01',  
                 'celery_tasks.task02'  
             ])  
  
cel.conf.timezone = 'Asia/Shanghai'  
cel.conf.enable_utc = False

# 使用命令给消息队列增加任务时
cel.conf.beat_schedule = {  
    # 名字随意  
    "add-every-10-seconds": {  
        # 执行task01下的send_email函数  
        "task": "celery_tasks.task01.send_email",  
        # 每隔两秒执行一次  
        # "schedule": 1.0,  
        # "schedule": crontab(minute='*/1'),        "schedule": timedelta(seconds=6),  # 推荐  
        # 参数  
        "args": ("张三",)  
    }  
}
```

task01和task02类似
```python title:celery_tasks/task01.py
import time  
from celery_tasks.celery import cel  
  
  
@cel.task  
def send_email(name):  
    print(f'向{name}发送邮件')  
    time.sleep(5)  
    return '发送邮件成功'
```

```python title:produce_task.py
from celery_tasks.task01 import send_email  
from celery_tasks.task02 import send_msg  
from datetime import datetime, timedelta  
  
# result = send_email.delay('xiaoming')  
# print(result)  
#  
# result = send_msg.delay('xiaohong')  
# print(result)  
  
# 定时执行任务  
# 方式一 使用datetime指定时间  
v1 = datetime(2024, 3, 9, 19, 40, 0)  
print(v1)  
v2 = datetime.utcfromtimestamp(v1.timestamp())  
print(v2)  
result = send_email.apply_async(args=('xiaoming',), eta=v2)  
print(result.id)  
  
# 方式二  
ctime = datetime.now()  
utc_time = datetime.utcfromtimestamp(ctime.timestamp())  
time_delay = timedelta(seconds=10)  
task_time = utc_time + time_delay  
result = send_email.apply_async(args=('xiaoming',), eta=task_time)  
print(result.id)
```


启动worker命令
```bash
# -----前台启动------
# 5.0之前
celery worker -A celery_tasks -l info -P eventlet
# 5.0版本后
celery -A celery_tasks worker -l info -P eventlet  #当前目录下应能发现celery_tasks这个包

# -----后台启动------
celery multi start w1 -A celery_tasks -l info    # w1 自定义名字

celery multi restart w1 -A celery_tasks -l info

celery multi stop w1 -A celery_tasks -l info

# 参数解析：
-A Celery对象所在包或文件
-l 日志等级
-P 执行器
-c 并发执行个数

# 使用命令向broker中添加任务
celery -A celery_tasks beat 
```
