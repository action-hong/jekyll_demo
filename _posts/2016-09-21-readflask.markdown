---
layout:     post
title:      "flask学习"
subtitle:   "关于app.route的实现"
date:       2016-09-21
author:     "kkopite"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - python
    - flask
---
> 最近看完了flask web开发,觉得很神奇,就开始撸源码了,先从最基本的route入手...

```
# 一个简单的路由
app.route('/')
def index():
    return 'Hello world'
```
   
   
``` 
# 看看route修饰器的构造
def route(self, rule, **options):
    def decorator(f):
        # 不管有没有自己定义endpoint, 先从options弹出来
        # 但是直接取出来,然后后面判断endpoint是None的话,加进去options就行了,为什么要这样呢??
        endpoint = options.pop('endpoint', None)
        self.add_url_rule(rule, endpoint, f, **options)
        return f
    return decorator
```
这里的`options`参数一直都是只用methods=["xx",,,]
默认是指监听"GET"方法以及隐式的"HEAD",options会被装化成底层的`werkzeug.routing.Rule`对象处理请求方法

```
# 看上面的代码可以得出add_url_rule其实就能达到注册效果
@app.route('/')
def index():
    pass
    
和下面代码是等价的
def index():
    pass
app.add_url_rule('/','index',index)
```

下面看看add_url_url的实现
```
def add_url_rule(self, rule, endpoint=None, view_func=None, **options):
        if endpoint is None:
            # endpoint为空的话,就endpoint = view_func.__name___
            endpoint = _endpoint_from_view_func(view_func)
        options['endpoint'] = endpoint
        methods = options.pop('methods', None)

        # 中间一串是对methods的处理,写得太清楚了
        # provide_automatic_options先从`view_func`中的attr取,没有的话在去看methods,methods如果有`OPTIONS`就为False,否则为True
        
        
        rule = self.url_rule_class(rule, methods=methods, **options)
        rule.provide_automatic_options = provide_automatic_options

        self.url_map.add(rule)      # endpoint和url的映射
        if view_func is not None:
            old_func = self.view_functions.get(endpoint)
            if old_func is not None and old_func != view_func:
                raise AssertionError('View function mapping is overwriting an '
                                     'existing endpoint function: %s' % endpoint)
            self.view_functions[endpoint] = view_func
            
        # 上面的代码看可以发现内部是通过endpoint映射view_func,
        # 即view_functions里面存放了endpoint->view_func的映射
```

可以看到url_map是关于endpoint和url的映射关系(怎么个关系还得看Rule的代码哈)
而view_functions是关于endpoint和url的映射关系

下面一串代码来瞧瞧url_map和view_functions

```
from flask import Flask

app = Flask(__name__)

@app.route('/',methods=['GET','POST'], endpoint='test')
def index():
	return 'hello world'

@app.route('/kkopite')
def get_some():
	return 'hello kkopite'

print(app.url_map)
print(app.view_functions)

#  url endpoint method 
Map([<Rule '/kkopite' (HEAD, GET, OPTIONS) -> get_some>,
 <Rule '/' (HEAD, GET, OPTIONS, POST) -> test>,
 <Rule '/static/<filename>' (HEAD, GET, OPTIONS) -> static>])
 
{'get_some': <function get_some at 0x0000020F5185E2F0>, 'test': <function index at 0x0000020F4F46C9D8>, 'static': <bound method _PackageBoundObject.send_static_file of <Flask 'read_route'>>}

```

然后再看run方法
```
def run(self, host=None, port=None, debug=None, **options):
    #省略
    from werkzeug.serving import run_simple
    # 即把自身当做一个wsgi应用传入
    run_simple(host, port, self, **options) 
    
# app实现wsgi的接口
def __call__(self, environ, start_response):
    return self.wsgi_app(environ, start_response)
    
def wsgi_app(self, environ, start_response):
    ctx = self.request_context(environ)         # RequestContext对象
    ctx.push()                                  # 压入_request_ctx_stack.top
    error = None
    try:
        try:
            response = self.full_dispatch_request()
        except Exception as e:
            error = e
            response = self.make_response(self.handle_exception(e))
        return response(environ, start_response)
    finally:
        if self.should_ignore_error(error):
            error = None
        ctx.auto_pop(error)

# 获取response
def full_dispatch_request(self):
    self.try_trigger_before_first_request_functions()   # 请求前执行函数(自己需求的话添加)
    try:
        request_started.send(self)
        rv = self.preprocess_request()                  # 执行@before_request修饰额函数
        if rv is None:                                  # 并无跳转..执行dispatch_request
            rv = self.dispatch_request()
    except Exception as e:
        rv = self.handle_user_exception(e)
    response = self.make_response(rv)
    response = self.process_response(response)
    request_finished.send(self, response=response)
    return response
    
def dispatch_request(self):
    # 取出之前push进去的ResponseContext对象(wsgi_app方法中)的request,里面关联了endpoint和args
    req = _request_ctx_stack.top.request    
    
    if req.routing_exception is not None:
        self.raise_routing_exception(req)
        
    # 取出当前url的rule
    rule = req.url_rule
    # if we provide automatic options for this URL and the
    # request came with the OPTIONS method, reply automatically
    if getattr(rule, 'provide_automatic_options', False) \
       and req.method == 'OPTIONS':
        return self.make_default_options_response()
    # otherwise dispatch to the handler for that endpoint
    
    # 找出对应的endpoint-->找出对应的view_func--->执行该方法
    return self.view_functions[rule.endpoint](**req.view_args)

```

## 总结
基本上大致流程就是这样了:
1. @app.route修饰器将指定的`url,denpoint,view_func`通过`Rule`,关联起来,其中`Rule`对象存放一组`url`,`methods`,`endpoint`的信息,放在url_map里面,然后`endpoint`和`view_func`组成映射关系存入view_fuctions中
2. app自己实现wsgi接口,将获取的environ作为参数实例一个RequestContext对象放到`_request_ctx_stack`
3. 然后在`dispatch_request`中将该对象取出,内部根据envrion获取url,url获取endpoint,enpoint找到view_func,完成从url到视图函数的过程(然而里面还挺多细节的呀)





