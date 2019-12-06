---
layout:     post
title:      "Celery实践的一些心得"
date:       2019-12-06 10:30:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - Python
---

### 前言
最近，公司项目使用Celery做异步任务踩了一些坑，后来阅读源码总结了一下经验。

### 一、理解task与request
通常我们会在celery注册实现的异步任务或定时任务task, 在celery worker运行时注册的task将会有一个全局化的实例。而每次调用某个task任务，实际上只是从broker获取到message创建一个新的worker.request.Request对象，并且将它赋值给对应task实例的`request`字段。所以，我们可以通过`task.request`获取每次task被调用的详情。

### 二、重试相关
```python
Class BaseTask(celery.Task)
	"""抽象出所有子类的共用行为"""
	def on_failure():
		pass
	def on_retry():
		pass
	def on_success():
		pass
	def after_return():
	    pass

@app.task(
    name=‘task.add’ # task名称，最佳实践是设置module.funcname，这样模块间不会冲突。
    autoretry_for=(Exception,),  # 注册需要重试的异常
    max_retries=2 # 最大重试次数
    soft_time_limit= 10 # 任务时间，到点raise SoftTimeLimitExceeded
    time_limit = 15 # 强制任务时间，到点关闭task(一般time_limit大于soft_time_limit)
    default_retry_delay = 1 # 重试时间间隔
    retry_backoff=True/1  # 设为1则第一次等待1s进行重试，第n次等待1*2^ns进行重试
    base=BaseTask
)
def add():
	add.name # 根据module和class名生成或者自己指定
	add.max_retries # 在装饰器中设置autoretry_for才有,超过max_retries会raise  MaxRetriesExceededError
	add.request.retries  # 重试次数
	add.request.id # task id
```

### 三、单例化资源
在Prefork模式中，常常需要每个子进程申请一个全局的资源句柄，而不是每次调用task时申请一次资源，例如Kafka producer、db Cursor等，通过下面的例子可以实现资源在进程内的单例化：
```python
from celery import Task

class DatabaseTask(Task):
    _db = None

    @property
    def db(self):
        if self._db is None:
            self._db = Database.connect()
        return self._db
that can be added to tasks like this:

@app.task(base=DatabaseTask)
def process_rows():
    for row in process_rows.db.table.all():
        process_row(row)
```