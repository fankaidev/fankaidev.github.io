---
layout: single
title:  "Draft"
---

[koa-passport](https://github.com/rkusa/koa-passport)是koa的一个中间件，它实际上只是对[passport](https://github.com/jaredhanson/passport)的一个封装。利用koa-passport可以简便的实现登录注册功能，不但包括本地验证，还有很多提供第三方登录的模块可以使用。


## 基本流程
`passport`的主要功能就是能够提供一个用户鉴权的框架，并把鉴权得到的用户身份供后续的业务逻辑来使用。而鉴权的具体过程是通过插件来实现，用户可以自己来实现，也可以使用已有的第三方模块实现各种方式的鉴权（包括OAuth或者OpenID)。

`passport`的代码还是有点复杂的，我们先看一个最简单的例子，然后逐步介绍后面的细节。

## 最小样例

下面是一个使用`koa-passport`的可以执行的最小样例

```javascript
const Koa = require('koa')
const app = new Koa()

// 定义一个验证用户的策略，需要定义name作为标识
const naiveStrategy = {
  name: 'naive',
  // 策略的主体就是authenticate(req)函数，在成功的时候返回用户身份，失败的时候返回错误
  authenticate: function (req) {
    let uid = req.query.uid
    if (uid) {
      // 策略很简单，就是从参数里获取uid，然后组装成一个user
      let user = {id: parseInt(uid), name: 'user' + uid}
      this.success(user)
    } else {
      // 如果找不到uid参数，认为鉴权失败
      this.fail(401)
    }
  }
}

// 调用use()来为passport新增一个可用的策略
const passport = require('koa-passport')
passport.use(naiveStrategy)
// 添加一个koa的中间件，使用naive策略来鉴权。这里暂不使用session
app.use(passport.authenticate('naive', {session: false}))

// 业务代码
const Router = require('koa-router')
const router = new Router()
router.get('/', async (ctx) => {
  if (ctx.isAuthenticated()) {
    // ctx.state.user就是鉴权后得到的用户身份
    ctx.body = 'hello ' + JSON.stringify(ctx.state.user)
  } else {
    ctx.throw(401)
  }
})
app.use(router.routes())

// server
const http = require('http')
http.createServer(app.callback()).listen(3000)

```

这段代码虽然没有实际意义，但是已经展示完整的鉴权流程和错误处理，运行结果如下
```
$ curl http://localhost:3000/\?uid\=128
hello {"id":128,"name":"user128"}%
$ curl http://localhost:3000
Unauthorized%  
``` 

可以看到这里鉴权的作用基本就是两个
1. 鉴权失败可以拒绝用户访问（也可以不做处理)
2. 鉴权成功会把用户记录到context

## 主流程
明白了上面的例子之后，我们就可以照着代码来看一下`passport`的主流程，定义在passport/authenticator.js文件中。

首先看看上面使用的`passport.use()`函数，其内容非常简单，就是把一个策略保存在本地，后续可以通过name来访问
```
Authenticator.prototype.use = function(name, strategy) {
  if (!strategy) {
    strategy = name;
    name = strategy.name;
  }
  if (!name) { throw new Error('Authentication strategies must have a name'); }
  
  this._strategies[name] = strategy;
  return this;
};
```

上面样例中使用的中间件`passport/authenticate()`，实际定义在passport/middleware/authenticate.js中，其返回值是一个函数，具体的实现如下，删减了部分代码，加了一下注释

```
module.exports = function authenticate(passport, name, options, callback) {
  // FK: 允许传入一个策略的数组
  if (!Array.isArray(name)) {
    name = [ name ];
    multi = false;
  }
  
  return function authenticate(req, res, next) {
    // FK: 省略部分代码...
    function allFailed() {
       // FK: 所有策略都失败后，如果设置了回调就调用，否则找到第一个failure，根据option进行flash/redirect/401
    }
    
    // FK: 这部分是主要逻辑
    (function attempt(i) {
      // FK: 尝试第i个策略
      var layer = name[i];
      if (!layer) { return allFailed(); }
    
      var prototype = passport._strategy(layer);  
      var strategy = Object.create(prototype);
   
      strategy.success = function(user, info) {
        // FK: 省略部分代码，用于记录flash/message

        // FK: 鉴权成功后会调用logIn()，把用户写入当前ctx
        req.logIn(user, options, function(err) {
          if (err) { return next(err); }          
          // FK: 成功跳转，代码比较复杂，省略了
      };
      
      strategy.fail = function(challenge, status) {
        // FK: 记录错误，使用下一个策略进行尝试，省略部分代码
        attempt(i + 1);
      };
      
      strategy.redirect = function(url, status) {
        // FK: 处理跳转
        res.statusCode = status || 302;
        res.setHeader('Location', url);
        res.setHeader('Content-Length', '0');
        res.end();
      };
      
      strategy.authenticate(req, options);
    })(0); // attempt
  };  
```
这部分代码就是`passport`的核心流程，其实就是定义好一些鉴权成功/失败/跳转等处理机制，然后就调用具体的策略进行鉴权。需要注意的是，虽然外面上面的样例中只传入了一个策略，但是其实`passport`支持同时使用多个策略，它会从头开始尝试各个策略，直到有一个策略做出处理或者已尝试所有策略为止。

### 小结
通过上面的样例和主流程的分析，大家应该能清楚`passport`大概做的事情是什么，也能够知道最基础的使用方式。

## session

在上面的例子中，我们没有使用session，因此每个请求都需要带上uid参数。实际使用中，一般会把鉴权后的用户身份会保存在cookie中供后续请求来使用。虽然passport并没有要求一定使用session，但其实是默认会使用session。

在上面的样例中，为了支持session，我们需要添加一些代码。

首先，我们需要在app中开启session支持，即使用`koa-session`
```
const session = require('koa-session')
app.keys = ['some secret']
const conf = {
  encode: json => JSON.stringify(json),
  decode: str => JSON.parse(str)
}
app.use(session(conf, app))
```

然后，因为我们的用户信息需要保留在session存储中(利用cookie或者服务端存储)，因此需要定义序列化和反序列的操作。下面的例子是一个示例。真实场景中，反序列化的时候肯定需要根据uid来检索真正的用户信息。
```
passport.serializeUser(function (user, done) {
  // 序列化的结果只是一个id
  done(NO_ERROR, user.id)
})

passport.deserializeUser(async function (str, done) {
  // 根据id恢复用户
  done(NO_ERROR, {id: parseInt(str), name: 'user' + str})
})
```

再然后，因为我们不需要naiveStrategy作为默认策略了，因此要把相应的use()语句去掉，转而只在用户明确要登录的时候才调用
```
router.get('/login',
  passport.authenticate('naive', { successRedirect: '/' })
)
```

最后，我们需要在app中开启`koa-passport`对session的支持

```
app.use(passport.initialize())
app.use(passport.session())
```

`initialzie()`函数的作用是只是简单为当前context添加passport字段，便于后面的使用。而
`passport.session()`则是`passport`自带的策略，用于从session中提取用户信息，其代码位于passport/strategies/session.js，内容如下

```
SessionStrategy.prototype.authenticate = function(req, options) {
  // FK: 确保已经初始化
  if (!req._passport) { return this.error(new Error('passport.initialize() middleware not in use')); }
  options = options || {};

  // FK: 从session中获取序列化后的user
  var self = this, su;
  if (req._passport.session) {
    su = req._passport.session.user;
  }

  if (su || su === 0) {
    // FK: 如果用户字段存在，调用自定义的反序列化函数来获取用户信息
    this._deserializeUser(su, req, function(err, user) {
      if (err) { return self.error(err); }
      if (!user) {
        delete req._passport.session.user;
      } else {
        var property = req._passport.instance._userProperty || 'user';
        req[property] = user;
      }
      self.pass();
      // FK: 省略
    });
  } else {
    // FK: 如果在session中找不到用户字段，直接略过
    self.pass();
  }
};
```

和我们上面自定义的naive策略类似，session策略的作用也即生成用户信息，只不过数据来源不是请求字段，而是session信息。

## Session Manager
在上面的主流程里，我们看到当鉴权成功时，会调用req.logIn()函数，其实还有logOut()和isAuthenticated(),都定义在passport/http/request.js。

其中logIn()和logOut()操作真正调用的是
SessionManager中的操作，其定义在passport/sessionmanager.js,主要流程如下：

```
function SessionManager(options, serializeUser) {
  // FK: ... 省略
  this._key = options.key || 'passport';
  this._serializeUser = serializeUser;
}

SessionManager.prototype.logIn = function(req, user, cb) {
  this._serializeUser(user, req, function(err, obj) {
    if (err) {
      return cb(err);
    }
    // FK: ... 省略
    req._passport.session.user = obj;
    // FK: ... 省略
    req.session[self._key] = req._passport.session;
    cb();
  });
}

SessionManager.prototype.logOut = function(req, cb) {
  if (req._passport && req._passport.session) {
    delete req._passport.session.user;
  }
  cb && cb();
}

module.exports = SessionManager;

```

Session Manager里定义了login和logout两个操作

* logIn()操作，会调用一个_serializeUser(), 然后把序列化的结果存到req.session['passport']。此时session的内容类似 ```{"passport":{"user":128},"_expire":1517357892908,"_maxAge":86400000}```
* logOut()操作更加简单，就是直接删除session里passport里的user字段。

此外，request还定义了isAuthenticated()函数，用于检查当前是否已经鉴权成功，代码如下
```
req.isAuthenticated = function() {
  var property = 'user';
  if (this._passport && this._passport.instance) {
    property = this._passport.instance._userProperty || 'user';
  }
  
  return (this[property]) ? true : false;
};
```

### 小结
至此，我们基本已经看完了`passport`的主要工作。`passport`之所以强大，在于他定义好了框架，但并没有确定具体的鉴权策略，用户可以根据需求来加入各种自定义的策略，现在已经有大量的模块可以使用了。

##passport-local

在上面的样例中，我们定义了自己的NaiveStrategy来实现对用户的鉴权，当然上面的代码毫无安全性可言。在真实环境中，最简单的鉴权一般是用户提交用户名和密码，然后服务端来校验密码，准确无误后才认为鉴权成功。

虽然这个过程可以通过扩展我们的NaiveStrategy来实现，不过我们已经有了`passport-local`这个库提供了一个本地鉴权的代码框架，可以直接使用。我们来看看其流程：

```
 // FK: passport-local 省略部分代码
function Strategy(options, verify) {
  // FK: 记录verify函数
  this._verify = verify;
  this._passReqToCallback = options.passReqToCallback;
}

Strategy.prototype.authenticate = function(req, options) {
  // 从req里找到username 和 pass
  options = options || {};
  var username = lookup(req.body, this._usernameField) || lookup(req.query, this._usernameField);
  var password = lookup(req.body, this._passwordField) || lookup(req.query, this._passwordField);
  
  if (!username || !password) {
    return this.fail({ message: options.badRequestMessage || 'Missing credentials' }, 400);
  }
  
  var self = this;
  
  function verified(err, user, info) {
    if (err) { return self.error(err); }
    if (!user) { return self.fail(info); }
    self.success(user, info);
  }
  
  // FK: 省略部分代码
  try {
    this._verify(username, password, verified);
  } catch (ex) {
    return self.error(ex);
  }
};
```

看代码其实流程也非常简单，就是自动从请求中获取`username`和`password`两个字段，然后提交给用户自定义的verify函数进行鉴权，然后处理鉴权的结果。

可以看到，这个框架做的事情其实很有限，主要的校验操作还是需要用户自己来定义，一个简单用法样例如下：
```javascript
const LocalStrategy = require('passport-local').Strategy
passport.use(new LocalStrategy(async function (username, password, done) {
  // FK: 根据username从数据库或者其他存储中拿到用户信息
  let user = await userStore.getUserByName(username)
  // FK: 把传入的password和数据库中存储的密码进行比较。当然这里不应该是明文，一般是加盐的hash值
  if (user && validate(password, user.hash)) {
    done(null, user)
  } else {
    log.info(`auth failed for`, username)
    done(null, false)
  }
}))
```

自此，我们看到用户只需要再定义一下用户的存储流程，基本上就可以实现一个简单的用户注册登录功能了。

## 后记
本文记录对`koa-passport`相关模块的代码学习过程，里边细节比较多，行文有些乱，还请见谅。如果大家只是想看使用方法，可以参考其他的文章，比如[这篇](https://segmentfault.com/a/1190000011557953)。

和之前看`koa-session`模块相比，`passport`模块没有使用async语法，就能明显感受到回调地狱的威力，代码一开始看的还是比较痛苦。总体感觉js还是过于灵活了，如果用来写业务，一定需要非常强健的工程规范才行。

文中两个样例的完整代码请参见 https://github.com/fankaigit/vauth/tree/master/server/sample
Hello github pages!
