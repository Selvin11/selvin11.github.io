---
title:  "Node.js + express + jade开发指南实例"
categories: Node.js
date:   2016-06-16 14:35:23 +0800
tags: Node.js
---


Node.js 开发指南实例——Microblog
<!-- more -->

1. app.js-----express详解及问题总结

*  自有模块&&引入模块

  Express 4.x 目前只有`express.static`一个唯一的中间件，用于托管内部静态资源，`express.static(path.join(__dirname, 'public'))`，此为常见用法，`__dirname`即为当前目录名。

  `http`/`fs`/`path`均为Node.js的原生模块

* 问题1 => flash()方法弃用

  Express 4.x 已经不支持`flash()`方法，需要引入模块`require('connect-flash')`，此模块就是提供原有的`flash()`方法，但在用之前需要`app.use(flash())`，这样相当于激活该方法，下面来讲讲`flash()`是什么，通过NPM和其对应的英文文档描述，主要起到处于客户端和服务端的桥梁作用，能够缓存服务器通过模板引擎传递给客户端的数据，并且是一次性的，属于用即弃的性质，因此简化的会话操作。

* 问题2 => mongodb相关模块的使用

  ```javascript
  var session = require('express-session');
  var settings = require('./settings');
  var MongoStore = require('connect-mongo')(session);
  ```
  这些都是相关的模块引入，主要是session的设置，这里引出关于session和cookie的关联问题，（cookie和session都是为了记录用户与浏览器之间以及浏览器和服务器之间的会话记录的，cookie存在于浏览器客户端中，session则存在服务器中，相比的话session的安全级别更高）。
  ​	
   引入模块之后，需要设置session，书中提供的方法已经不适用了，解决如下：
   注释掉db,host,port等一个个的设置，直接写上真正的url地址即可
   
    ```javascript
        app.use(session({
            resave:false,//添加这行  
            saveUninitialized: true,//添加这行  
            secret: settings.cookieSecret,
            key: settings.db,//cookie name
            cookie: {maxAge: 1000 * 60 * 60 * 24 * 30},//30 days
            store: new MongoStore({
            //现在需要直接写上url地址，其余的都注释掉就行
            // db: settings.db,
            // host: settings.host
            url:'mongodb://localhost/nodeweb'
            })
        }));
    ```
*   问题3 => view视图交互代码弃用

   关于视图交互，即MVC中的"V"，主要用作用是为用户提供更好的视图体验以及缓存数据信息，打通数据库与浏览器之间的管理，比如，用户在登录时，重名，密码错误等问题，这些都可以通过session先将信息传递给response服务器返回信息，然后通过本地的flash()方法，将服务器返回的信息，通过模板引擎二次加工，呈现给用户，提升用户体验。

   解决代码如下，替换原有的即可：

  ```javascript
  app.use(function(req,res,next){
      res.locals.user = req.session.user;
      var err = req.flash('error');
      var success = req.flash('success');
      res.locals.error = err.length ? err : null;
      res.locals.success = success.length ? success : null;
      //next()函数相当于跳过继续，实质是将视图路由控制权再交还给其他路由视图
  next();
  });		 	
  ```

*   问题4 => 路由

     这个其实是理解的关键，express本身就是路由 + 中间件（相当于第三方模块）构成的，但是其控制路由的方式方法很多，就有种jquery中ajax()一整套方法以及其各种分支方法的组合，先从书中案列讲起，书中一开始提及的就是`app`，即`var app = express();`，app相当于express的实例，从官方文档可以了解，挂载了许多方法，其中就有一个`app.route()`，而且express本身就有一个`Router`模块，（区别是什么：本身的模块也可以控制设置路由，但是是挂载在express.Router()下的，无法使用express()下的方法，所以一般是在express.Router()下配置路由，再导出模块，使用express()的use方法调用）这个方法下面可以链式调用get/post...一系列方法。
     源码解析：

      ```javascript
      //首先是app.config()配置
      var routes = require('./routes/index');
      app.use(routes);
      //这个配置就是路由调用方法的关键，routes是存放在routes文件夹下面的index.js模块中的
      //于是我们可以将路由控制先写在index.js中，然后让app.js调用
      ```

      但是，以上代码已经不适用，解决方案如下，还是讲路由控制的代码挂载在index.js模块中。代码如下：

      ```javascript
      	var express = require('express');
      	var router = express.Router();
      	var crypto = require('crypto');
      	var User = require('../models/user.js');
      	var Post = require("../models/post.js");
      	// 定义网站的各项路由，然后输出此模块
      	/* GET home page. */
      	router.get('/', function(req, res, next) {
      		// 通过Post的get方法中的回调函数获取posts数据
      		Post.get(null,function (err,posts) {
      			if (err) {
      				posts=[];
      			}
      			res.render('index',{
      				title: '首页',
      				posts: posts
      			})
      		})
      	});
      ```

      这只是其中主页面的路由设置信息，主要就是用express.Router()来配置路由，然后导出路由模块，由express()来写入session信息，从而使得数据库与浏览器客户端建立连接，这样也使得数据和路由主配置较为干净，即app.js较为清晰，能一眼看到引入的模块，以及数据库信息。

      ```javascript
      var express = require('express');
      var path = require('path');
      // 数据库及session相关模块引入
      var session = require('express-session');
      var settings = require('./settings');
      var MongoStore = require('connect-mongo')(session);
      var User = require('./models/user.js');
      var Post = require('./models/post.js');
      var favicon = require('serve-favicon');
      var logger = require('morgan');
      var cookieParser = require('cookie-parser');
      var bodyParser = require('body-parser');
      var crypto = require('crypto');
      var flash = require('connect-flash');
      var routes = require('./routes/index');
      var app = express();
      // 添加flash()方法==================================要点1
      app.use(flash());
      // view engine setup
      app.set('views', path.join(__dirname, 'views'));
      app.set('view engine', 'jade');
      // uncomment after placing your favicon in /public
      //app.use(favicon(path.join(__dirname, 'public', 'favicon.ico')));
      app.use(logger('dev'));
      app.use(bodyParser.json());
      app.use(bodyParser.urlencoded({ extended: false }));
      app.use(cookieParser());
      app.use(express.static(path.join(__dirname, 'public')));
      // mongodb connect
      app.use(session({
        // ==============================================要点2
        resave:false,//添加这行  
        saveUninitialized: true,//添加这行  
        secret: settings.cookieSecret,
        key: settings.db,//cookie name
        cookie: {maxAge: 1000 * 60 * 60 * 24 * 30},//30 days
        store: new MongoStore({
          // db: settings.db,
          // host: settings.host
          // =============================================要点3
          url:'mongodb://localhost/nodeweb'
        })
      }));
      // view 视图交互，实现不同登录状态下呈现不同内容
      // 设置请求头user信息，为index.jade做铺垫
      // ==================================================要点4
      app.use(function(req,res,next){
        res.locals.user = req.session.user;
        var err = req.flash('error');
        var success = req.flash('success');
        res.locals.error = err.length ? err : null;
        res.locals.success = success.length ? success : null;
        next();
      });
      // 路由输出
      app.use('/',routes);
      // catch 404 and forward to error handler
      app.use(function(req, res, next) {
        var err = new Error('Not Found');
        err.status = 404;
        next(err);
      });
      // error handlers
      // development error handler
      // will print stacktrace
      if (app.get('env') === 'development') {
        app.use(function(err, req, res, next) {
          res.status(err.status || 500);
          res.render('error', {
            message: err.message,
            error: err
          });
        });
      }
      // production error handler
      // no stacktraces leaked to user
      app.use(function(err, req, res, next) {
        res.status(err.status || 500);
        res.render('error', {
          message: err.message,
          error: {}
        });
      });
      module.exports = app;
      ```
> Jade

通过我们建立的post/user/..等模型中的回调函数将模板中的数据信息写入数据库，以及通过回调函数拿到数据库中的信息传递给模板，并让浏览器解析产生页面。
 主要是模板中的继承extend/包含include/自定义代码块blcok...等的使用。

 由于书中采用的是ejs模板，由于和后端代码书写太相像，使用过便放弃了，使得html代码看着太奇怪，jade的简写其实很好，就是有时候没注意空格和缩进=。=
 贴一个将文中一个ejs模板替换的jade模板，以供参考：

   ```html
   .row
   each item in posts
      .col-md-4	  
          h2
              a(href='/u/'+item.user)= item.user
                  != '<samll>'+' 说'+'</samll>'        
          p!= '<samll>'+item.time+'</samll>'
          p= item.post
   ```

 以上是从Node.js开发指南中的总结以及问题处理，感谢作者，感谢百度及谷歌，问题不可怕，先自己多想，多看官方文档，实在不行，还有N多大神在网上写有解答，之前也用node仿照慕课视频上做了一个电影网站，但感觉不系统，这次便重新系统了巩固一下，其实Node.js很够学，只是会使用require几个模块是没什么大用的，主要是要学习其机制，模块，MVC，模板这些在早些年，在其它语言中都有过，只是将业务逻辑，视图，行为分离，使得人能有一个大局观，然后在各个层中专一，逐层击破罢了，如果人能够轻松实现多线程，也就没有单线程非阻塞这一说了~

 > 鄙人小白一个，从心喜欢前端，仍在不断深入前端领域以及强化基础，从未指望一蹴而就，只想深入几十载，再回首。