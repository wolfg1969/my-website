+++
Categories = []
Description = ""
Tags = ["技术", "Python", "Django", "Django REST Framework"]
date = "2016-04-20T17:06:03+08:00"
isCJKLanguage = true
slug = "django-rest-framework-custom-exception-handling"
title = "定制 Django REST Framework 的异常处理"

+++

今天借助修复其它 Bug 的机会，对项目中 REST API 的异常处理机制进行了优化。

项目的 REST API 是基于 Django REST Framework 实现的，之前已经按照其文档的说明，对 EXCEPTION_HANDLER 进行了定制，用统一的 JSON 数据格式来输出异常错误信息，包括 500 错误。没有定制时，对于系统所有未特殊处理的异常，Django REST Framework 的处理方法是继续向上抛出，而不是像其它情况那样处理成自己封装的 Exception Response。这样的话，客户端得到的响应是 Django 框架的 500 错误页面，不便于客户端的统一处理。

自定义的异常处理函数如下：

{{< highlight python "linenos=inline,hl_lines=5 8 13" >}}
from rest_framework.views import exception_handler

def my_api_exception_handler(exc):
    # Call REST framework's default exception handler first to get the standard error response.
    response = exception_handler(exc)

    # handle 500 errors
    if response is None:
        if isinstance(exc, APIException):
            detail = exc.detail
        else:
            detail = exc.message if settings.DEBUG else 'Internal Server Error'
        response = Response({'detail': detail}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)

    return response
    
{{< /highlight >}}

这里先调用框架默认的异常处理函数（Line 5），当这个函数返回的 response 对象为 None 时（Line 8），说明异常没有被它处理。这时我们就自己生成一个 status 状态码为 500 的 Response 对象（Line 13），而不是返回 None 。

自定义的异常处理函数必须在 settings 里启用，如下
``` python
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'my_project.my_app.utils. my_api_exception_handler'
}
```

今天的改进是对异常处理函数中日志的改进。原来的日志记录在发生 500 的错误，就记录一条 Error 日志：
``` python
logger.error("api view 500 error: %s >>>\n %s", str(exc), traceback.format_exc())
```
在生产环境下，这条日志会产生一封通知邮件，但效果并不好，异常堆栈变成了一长串字符被塞进了邮件的主题里，也没有当前 Request 的上下文信息，很难从中分析定位错误。

改进后的日志语句被挪到了 APIView 的 handle_exception 方法里：
{{< highlight python "linenos=inline,hl_lines=5 8 13" >}}
def handle_exception(self, exc):
    resp = super(GenericAPIView, self).handle_exception(exc)
    if resp.status_code == status.HTTP_500_INTERNAL_SERVER_ERROR:
        logger.error("API View 500 error: %s", self.request.path,
                     exc_info=sys.exc_info(),
                     extra={
                         'status_code': 500,
                         'request': self.request
                     })
    return resp
    
{{< /highlight >}}
可以看到，异常信息和当前请求信息都被记录了（ Line 5 和 Line 8）。exc_info 和 extra 参数的说明见 Python 标准库文档，这里也参考了 Django 中 AdminEmailHandler 类的实现。

以上。
