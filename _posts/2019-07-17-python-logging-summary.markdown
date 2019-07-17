---
layout:     post
title:      "Python标准库logging"
date:       2019-07-17 18:11:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - Python
---

## 0.logging库的结构

## 1.线程锁
logging模块可能被用于多线程中，模块中有许多全局变量，如果不加锁也会导致并发访问数据的安全问题。logging源码中表明：在使用多线程模块threading时，提供多线程重入锁的获取和释放方法_acquireLock, _releaseLock:

```python
try:
    import threading
except ImportError: #pragma: no cover
    threading = None

if threading:
    _lock = threading.RLock()
else: #pragma: no cover
    _lock = None

def _acquireLock():
    """
    Acquire the module-level lock for serializing access to shared data.

    This should be released with _releaseLock().
    """
    if _lock:
        _lock.acquire()

def _releaseLock():
    """
    Release the module-level lock acquired by calling _acquireLock().
    """
    if _lock:
        _lock.release()
```

关于可重入锁RlLock:

> RLock允许多次加锁，但释放的次数必须与加锁次数相同。

其应用场景主要是:
> 对于线程处理中一些比较复杂的代码逻辑过程，比如很多层的函数调用，而这些函数其实都需要进行加锁保护数据访问。如果用普通的Lock，当一个函数A对某个数据加锁，它内部调用另一个函数B，如果B内部也对同一个数据加锁，那么这种情况就也会导致死锁，而RLock可以解决这个问题。

## 2.LogRecord

源码中对`LogRecord`类的描述是`A LogRecord instance represents an event being logged.`，LogRecord对象代表一条日志事件的记录。它的初始化方法是:
```python
def __init__(self, name, level, pathname, lineno,
                 msg, args, exc_info, func=None, sinfo=None, **kwargs):
        """
        Initialize a logging record with interesting information.
        """
        ct = time.time()
        self.name = name
        self.msg = msg
        #
        # The following statement allows passing of a dictionary as a sole
        # argument, so that you can do something like
        #  logging.debug("a %(a)d b %(b)s", {'a':1, 'b':2})
        # Suggested by Stefan Behnel.
        # Note that without the test for args[0], we get a problem because
        # during formatting, we test to see if the arg is present using
        # 'if self.args:'. If the event being logged is e.g. 'Value is %d'
        # and if the passed arg fails 'if self.args:' then no formatting
        # is done. For example, logger.warning('Value is %d', 0) would log
        # 'Value is %d' instead of 'Value is 0'.
        # For the use case of passing a dictionary, this should not be a
        # problem.
        # Issue #21172: a request was made to relax the isinstance check
        # to hasattr(args[0], '__getitem__'). However, the docs on string
        # formatting still seem to suggest a mapping object is required.
        # Thus, while not removing the isinstance check, it does now look
        # for collections.Mapping rather than, as before, dict.
        if (args and len(args) == 1 and isinstance(args[0], collections.Mapping)
            and args[0]):
            args = args[0]
        self.args = args
        self.levelname = getLevelName(level)
        self.levelno = level
        self.pathname = pathname
        try:
            self.filename = os.path.basename(pathname)
            self.module = os.path.splitext(self.filename)[0]
        except (TypeError, ValueError, AttributeError):
            self.filename = pathname
            self.module = "Unknown module"
        self.exc_info = exc_info
        self.exc_text = None      # used to cache the traceback text
        self.stack_info = sinfo
        self.lineno = lineno
        self.funcName = func
        self.created = ct
        self.msecs = (ct - int(ct)) * 1000
        self.relativeCreated = (self.created - _startTime) * 1000
        if logThreads and threading:
            self.thread = threading.get_ident()
            self.threadName = threading.current_thread().name
        else: # pragma: no cover
            self.thread = None
            self.threadName = None
        if not logMultiprocessing: # pragma: no cover
            self.processName = None
        else:
            self.processName = 'MainProcess'
            mp = sys.modules.get('multiprocessing')
            if mp is not None:
                # Errors may occur if multiprocessing has not finished loading
                # yet - e.g. if a custom import hook causes third-party code
                # to run when multiprocessing calls import. See issue 8200
                # for an example
                try:
                    self.processName = mp.current_process().name
                except Exception: #pragma: no cover
                    pass
        if logProcesses and hasattr(os, 'getpid'):
            self.process = os.getpid()
        else:
            self.process = None

    def __str__(self):
        return '<LogRecord: %s, %s, %s, %s, "%s">'%(self.name, self.levelno,
            self.pathname, self.lineno, self.msg)

    __repr__ = __str__

    def getMessage(self):
        """
        Return the message for this LogRecord.

        Return the message for this LogRecord after merging any user-supplied
        arguments with the message.
        """
        msg = str(self.msg)
        if self.args:
            msg = msg % self.args
        return msg
```
从__init__方法可以看出，它实例化了一条日志的对象，包含了所需的所有信息字段：日志名，日志消息，日志等级，调用处的模块名，调用处的函数名，调用处的行号，调用栈信息，执行信息等，甚至还有打印该日志的线程名，进程名。

```python
#
#   Determine which class to use when instantiating log records.
#
_logRecordFactory = LogRecord

def setLogRecordFactory(factory):
    """
    Set the factory to be used when instantiating a log record.

    :param factory: A callable which will be called to instantiate
    a log record.
    """
    global _logRecordFactory
    _logRecordFactory = factory

def getLogRecordFactory():
    """
    Return the factory to be used when instantiating a log record.
    """

    return _logRecordFactory

def makeLogRecord(dict):
    """
    Make a LogRecord whose attributes are defined by the specified dictionary,
    This function is useful for converting a logging event received over
    a socket connection (which is sent as a dictionary) into a LogRecord
    instance.
    """
    rv = _logRecordFactory(None, None, "", 0, "", (), None, None)
    rv.__dict__.update(dict)
    return rv
```

从上面我们知道，LogRecord可以用来提供实例化日志记录，但它其实是官方提供的一个实例化日志记录的工厂类，我们可以通过setLogRecordFactory方法自定义工厂类，来实现控制任意我们想要修改的日志字段。


## 3.Filter

源码对于Filter类的描述是`Filter instances are used to perform arbitrary filtering of LogRecords.`，Filter用来对一条LogRecord执行任意规则的过滤。

## 4.Filterer

源码对于Filterer类的描述是`A base class for loggers and handlers which allows them to share
    common code.`，一个为handlers和loggers提供共同代码的基类。

它包含了三个方法addFilter，removeFilter，filter，不难看出来，它主要为handler提供了对`log record`过滤器的添加，移除以及过滤方法。需要注意的是: filter进行过滤，决定一条`log record`是被logged还是被dropped时，实行的是一票否决制，即任何一个filter返回零值时表示该record将被丢弃。

## 5.Handler
    NullHandler
    StreamHandler
    FileHandler

## 6.Logger
    RootLogger

## 7.LoggerAdapter

## 参考

- [1][logging-cookbook](https://docs.python.org/3/howto/logging-cookbook.html)

- [2][rlock的实际用途](https://ask.csdn.net/questions/709872)
