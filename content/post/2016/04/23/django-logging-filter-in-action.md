+++
Categories = []
Description = ""
Tags = ["技术", "Python", "Django", "Logging"]
date = "2016-04-23T10:36:22+08:00"
isCJKLanguage = true
slug = "django-logging-filter-in-action"
title = "Django 日志过滤器实战"

+++

Django 的日志配置采用标准的 Python 日志配置方式，默认采用 dictConfig 的格式。Python 文档里对日志过滤器 Filter 的解释是一种控制日志记录 Record 是否输出的机制。就像物理世界中的筛子，只有特定大小或形状的物体才能通过，Filter 可以按照特定的需求“筛选”出我们需要的日志记录。

我们来看看 Django 的一个内置过滤器 ```RequireDebugFalse``` 。这个过滤器一般是用于发送错误邮件日志，只有设置了 ```DEBUG = False```时才把错误日志发送给管理员（生产环境，避免邮件轰炸） ：
{{<highlight python "hl_lines=2 3 9">}}
'filters': {
    'require_debug_false': {
        '()': 'django.utils.log.RequireDebugFalse',
    }
},
'handlers': {
    'mail_admins': {
        'level': 'ERROR',
        'filters': ['require_debug_false'],
        'class': 'django.utils.log.AdminEmailHandler'
    }
},
{{</highlight>}}

可以看出，过滤器的配置方法是在 filters 的 ```dict``` 里指定一个别名 ```'require_debug_false'```作为 key，value 又是一个 ```dict```。这个 ```dict``` 用来描述过滤器如何被实例化。其中，```'()'```作为特殊的 key 用来指定过滤器是哪个类的实例（见 ```logging.config.DictConfigurator```）。其它的 key 用来指定实例化过滤器是传给 ```__init__``` 方法的参数。

再来看 ```RequireDebugFalse``` 过滤器的实现：

{{< highlight python "hl_lines=2" >}}
class RequireDebugFalse(logging.Filter):
    def filter(self, record):
        return not settings.DEBUG
{{< /highlight >}}

很简单，就是定义 ```filter``` 方法，日志记录 record 作为唯一参数，返回值为 ```0``` 或 ```False``` 表示日志记录将被抛弃，```1``` 或 ```True```则表示记录别放过。

过滤器不仅可以用来滤除不需要的日志记录，还可以修改日志记录。例如，我们可以给日志记录添加自定义的属性，用来区分日志来自于哪个应用和运行环境。

{{<highlight python "hl_lines=3 4 7 8">}}
class StaticFieldFilter(logging.Filter):

    def __init__(self, fields):
        self.static_fields = fields

    def filter(self, record):
        for k, v in self.static_fields.items():
            setattr(record, k, v)
        return True
 {{</highlight>}}
 
 这个过滤器把初始化时得到的属性键值对设为每条日志记录的属性，用法如下：
{{<highlight python "hl_lines=7 8 9 10">}}
'filters': {
    'require_debug_false': {
        '()': 'django.utils.log.RequireDebugFalse',
    },
    'static_fields': {
        '()': 'example.utils.log.StaticFieldFilter',
        'fields': {
            'project': os.environ['PROJECT_NAME'],
            'environment': os.environ['ENV_NAME'],
        },
    }
},
'handlers': {
    'mail_admins': {
        'level': 'ERROR',
        'filters': ['require_debug_false', 'static_fields'],
        'class': 'django.utils.log.AdminEmailHandler'
    }
},
{{</highlight>}}

注意 ```fields``` 是如何配置传入的。

更进一步，我们还可以让错误日志邮件的标题也包含自定义的属性：
{{<highlight python "hl_lines=4">}}
def filter(self, record):
    for k, v in self.static_fields.items():
        setattr(record, k, v)
        record.levelname = '[%s] %s' % (v, record.levelname)
    return True
{{</highlight>}}

再来看一个例子，Django 可以把 ```request``` 请求信息设为日志记录的属性，但有些日志处理器（比如 Graylog2 的 GELFHandler ）不能序列化 Request 对象，需要做些处理：
{{<highlight python "hl_lines=5 6">}}
from django.http import build_request_repr
class RequestFilter(logging.Filter):

    def filter(self, record):
        if hasattr(record, 'request'):
            record.request_repr = build_request_repr(record.request)
            del record.request
        return True
{{</highlight>}}
这里我们把 ```request``` 对象用 Django 内置的函数 ```build_request_repr ``` 转为字符串格式，然后删掉日志记录的 ```request``` 属性。

配置方法：
{{<highlight python "hl_lines=12 13 14 25">}}
'filters': {
    'require_debug_false': {
        '()': 'django.utils.log.RequireDebugFalse',
    },
    'static_fields': {
        '()': 'example.utils.log.StaticFieldFilter',
        'fields': {
            'project': os.environ['PROJECT_NAME'],
            'environment': os.environ['ENV_NAME'],
        },
    },
    'exclude_django_request': {
        '()': 'example.utils.log.RequestFilter',
    }
},
'handlers': {
    'gelf': {
        'level': 'DEBUG',
        'class': 'graypy.GELFHandler',
        'host': os.environ['GRAYLOG2_HOST'],
        'port': int(os.environ['GRAYLOG2_PORT']),
        'filters': [
            'require_debug_false', 
            'static_fields', 
            'exclude_django_request', 
        ],
    },
},

{{</highlight>}}

参考:

* https://docs.python.org/2/library/logging.html
* https://docs.djangoproject.com/en/1.9/topics/logging/
* [Central logging in Django with Graylog2 and graypy](https://www.caktusgroup.com/blog/2013/09/18/central-logging-django-graylog2-and-graypy/)