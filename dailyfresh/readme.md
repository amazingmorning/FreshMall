
# python异步模块celery 
## 一、Celery介绍和基本使用 
Celery 是一个 基于python开发的分布式异步消息任务队列，通过它可以轻松的实现任务的异步处理， 如果你的业务场景中需要用到异步任务，就可以考虑使用celery， 举几个实例场景中可用的例子:

你想对100台机器执行一条批量命令，可能会花很长时间 ，但你不想让你的程序等着结果返回，而是给你返回 一个任务ID,你过一段时间只需要拿着这个任务id就可以拿到任务执行结果， 在任务执行ing进行时，你可以继续做其它的事情。 
你想做一个定时任务，比如每天检测一下你们所有客户的资料，如果发现今天 是客户的生日，就给他发个短信祝福
 

Celery 在执行任务时需要通过一个消息中间件来接收和发送任务消息，以及存储任务结果， 一般使用rabbitMQ or Redis,后面会讲

#### 1.1 Celery有以下优点：

- 简单：一单熟悉了celery的工作流程后，配置和使用还是比较简单的
- 高可用：当任务执行失败或执行过程中发生连接中断，celery 会自动尝试重新执行任务
- 快速：一个单进程的celery每分钟可处理上百万个任务
- 灵活： 几乎celery的各个组件都可以被扩展及自定制

![enter description here](./images/1554623860799.png)

celery名词：

- 任务task：就是一个Python函数。
- 队列queue：将需要执行的任务加入到队列中。
- 工人worker：在一个新进程中，负责执行队列中的任务。
- 代理人broker：负责调度，在布置环境中使用redis。
#### 1.2 Celery安装使用
Celery的默认broker是RabbitMQ, 仅需配置一行就可以

1
broker_url = 'amqp://guest:guest@localhost:5672//'
rabbitMQ 没装的话请装一下，安装看这里  http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html#id3

 

使用Redis做broker也可以

安装redis组件


``` shell
$ pip install -U "celery[redis]"
```

配置

``` vhdl
Configuration is easy, just configure the location of your Redis database:

app.conf.broker_url = 'redis://localhost:6379/0'
Where the URL is in the format of:

redis://:password@hostname:port/db_number
all fields after the scheme are optional, and will default to localhost on port 6379, using database 0.
```

如果想获取每个任务的执行结果，还需要配置一下把任务结果存在哪

If you also want to store the state and return values of tasks in Redis, you should configure these settings:

app.conf.result_backend = 'redis://localhost:6379/0'

#### 1.3 开始使用Celery
安装celery模块

``` shell
$ pip install celery
```

创建一个celery application 用来定义你的任务列表
创建一个任务文件就叫tasks.py吧

``` python
from celery import Celery
 
app = Celery('tasks',
             broker='redis://localhost',
             backend='redis://localhost')
 
@app.task
def add(x,y):
    print("running...",x,y)
    return x+y
```

启动Celery Worker来开始监听并执行任务

``` shell
$ celery -A tasks worker --loglevel=info
```

调用任务

再打开一个终端， 进行命令行模式，调用任务　　

``` python
>>> from tasks import add
>>> add.delay(4, 4)
```

看你的worker终端会显示收到 一个任务，此时你想看任务结果的话，需要在调用 任务时　赋值个变量

``` livecodeserver
>>> result = add.delay(4, 4)
The ready() method returns whether the task has finished processing or not:

>>> result.ready()
False
You can wait for the result to complete, but this is rarely used since it turns the asynchronous call into a synchronous one:

>>> result.get(timeout=1)
In case the task raised an exception, get() will re-raise the exception, but you can override this by specifying the propagate argument:

>>> result.get(propagate=False)
If the task raised an exception you can also gain access to the original traceback:

>>> result.traceback
…
```
### 与Django项目结合
django 可以轻松跟celery结合实现异步任务，只需简单配置即可
比如实现一个异步发送激活邮件的例子：
第一步：在项目的文件下面创建一个python 文件包：selery_tasks。在下面创建一个task.py文件，用来处理耗时需要异步处理的task

``` python
import os
# import django
# os.environ.setdefault("DJANGO_SETTINGS_MODULE", "dailyfresh.settings")
# django.setup()

from goods.models import GoodsType,IndexGoodsBanner,IndexPromotionBanner,IndexTypeGoodsBanner
from django_redis import get_redis_connection

# 创建一个Celery类的实例对象
app = Celery('celery_tasks.tasks', broker='redis://172.16.179.142:6379/8')


# 定义任务函数
@app.task
def send_register_active_email(to_email, username, token):
    '''发送激活邮件'''
    # 组织邮件信息
    subject = '天天生鲜欢迎信息'
    message = ''
    sender = settings.EMAIL_FROM
    receiver = [to_email]
    html_message = '<h1>%s, 欢迎您成为天天生鲜注册会员</h1>请点击下面链接激活您的账户<br/><a href="http://127.0.0.1:8000/user/active/%s">http://127.0.0.1:8000/user/active/%s</a>' % (username, token, token)

    send_mail(subject, message, sender, receiver, html_message=html_message)
    time.sleep(5)

```
第二步：在对于的视图函数中调用：

``` python
# 发邮件
send_register_active_email.delay(email, username, token)
```

第三步：
启动Redis，如果已经启动则不需要启动。

``` ebnf
sudo service redis start
```
第四步：
启动worker。

``` shell
python manage.py celery worker --loglevel=info
```
