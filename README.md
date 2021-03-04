# node.js-blog
手把手教你用 node.js 打造个人blog

@[TOC]
# 项目
## 展示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210303181010324.gif#pic_center)
## 链接
[https://download.csdn.net/download/weixin_45525272/15545983](https://download.csdn.net/download/weixin_45525272/15545983)

GitHub今天上不去，就没往上发，过几天补上
# 实现讲解
## 项目目录
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210304103316324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NTUyNTI3Mg==,size_16,color_FFFFFF,t_70)

首先要npm所需要的包
## 1. npm
```shell
npm init -y

npm install art-template blueimp-md5 body-parser bootstrap express express-art-template express-session jquery mongoose
```
## 2.功能讲解
### app.js讲解
在app.js中我们导入必要的模块并进行必要配置，
express框架，path等系统模块，body-parser等中间件

```javascript
var express = require('express')
var path = require('path')
var bodyParser = require('body-parser')
var session = require('express-session')
var router = require('./router')

var app = express();

app.use('/public/',express.static(path.join(__dirname,'./public')));
app.use('/node_modules/',express.static(path.join(__dirname,'./node_modules')));

// 模板引擎 art-template
app.engine('html',require('express-art-template'))
app.set('views',path.join(__dirname,'./views/'));

// 配置解析表单 POST 请求体插件（注意：一定要在 app.use(router) 之前 ）
// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }))
// parse application/json
app.use(bodyParser.json())

app.use(session({
  // 配置加密字符串，它会在原有加密基础之上和这个字符串拼起来去加密
  // 目的是为了增加安全性，防止客户端恶意伪造
  secret: 'itcast',
  resave: false,
  saveUninitialized: false // 无论你是否使用 Session ，我都默认直接给你分配一把钥匙
}))

```
将各种请求写到router.js中，在app.js中需要导入第三方模块

```javascript
var router = require('./router')

... 引入其他模块，进行配置 ....

// 把路由挂载到 app 中
app.use(router);
```
最后我们监听3000端口进行开启 node.js 服务

```javascript
app.listen(3000,function(){
    console.log('server is running.....');
})
```
当在浏览器中进行请求时，会进入到 router.js（路由器） 中进行各种请求的判断（这个项目请求与功能还是比较少的，就没有再将router进拆分，都放在了 router.js 中）
### router.js讲解
#### 请求说明

| 请求名            | 请求类型 | 请求参数                  | 请求页面                | 请求说明                                                     |
| ----------------- | -------- | ------------------------- | ----------------------- | ------------------------------------------------------------ |
| /                 | get      |                           | index.html              | 进入博客主页面                                               |
| /register         | get      |                           | register.html           | 进入注册页面                                                 |
| /register         | post     | email，nickname，password | 注册成功：index.html    | 注册请求：邮件，昵称（二者均唯一），密码，成功自动登录到index.html页面 |
| /login            | get      |                           | login.html              | 进入登陆页面                                                 |
| /login            | post     | email，password           | 注册成功：index.html    | 登陆请求：邮件，密码，成功自动登录到index.html页面           |
| /settings/profile | get      |                           | ./settings/profile.html | 进入个人信息编辑页面                                         |
| /settings/admin   | get      |                           | ./settings/admin.html   | 进入个人管理页面                                             |
| /logout           | get      |                           | login.html              | 退出登陆请求，请求成功退出当前账号，并进入登陆页面           |

### get
像各种`get`请求的处理是比较简单的，直接render(界面，{参数}) 就可以了（下面代码中 req.session.user 是请求中的 session，在 `/login` 请求中进行了 对 session 的设置，参考下面的 post 讲解）
```javascript
var express = require('express')
var User = require('./models/user')
var md5 = require('blueimp-md5')

var router = express.Router()

router.get('/', function (req, res) {
  // console.log(req.session.user)
  res.render('index.html', {
    user: req.session.user
  })
})

router.get('/login', function (req, res) {
  res.render('login.html')
})


router.get('/register', function (req, res) {
  res.render('register.html')
})



router.get('/settings/profile', function (req, res) {
  res.render('./settings/profile.html',{
    user: req.session.user
  })
})


router.get('/settings/admin',function(req,res){
  res.render('./settings/admin.html',{
    user: req.session.user
  })
})


router.get('/logout', function (req, res) {
  // 清除登陆状态
  req.session.user = null

  // 重定向到登录页
  res.redirect('/login')
})
```

### post
#### /login
登陆功能验证包括三个步骤
1. 获取表单数据
2. 查询数据库用户名密码是否正确
3. 发送响应数据


登陆请求中，首先我们要查找是否有这个用户

```javascript
 User.findOne({
    email: body.email,
    password: md5(md5(body.password))
  }, function (err, user) {
    if (err) {
      return res.status(500).json({
        err_code: 500,
        message: err.message
      })
    }
 });
```

如果邮箱和密码匹配，则 user 是查询到的用户对象，否则就是 null
```javascript
if (!user) {
  return res.status(200).json({
    err_code: 1,
    message: 'Email or password is invalid.'
  })
}
```
用户存在，登陆成功，通过 Session 记录登陆状态
```javascript
req.session.user = user

res.status(200).json({
  err_code: 0,
  message: 'OK'
})
```
#### /register
注册和登陆的基本流程（上文三步）还是差不多的

首先也是查找是否存在（这里我们用 mongodb 或 语句），避免重复注册
```javascript
  User.findOne({
    $or: [{
        email: body.email
      },
      {
        nickname: body.nickname
      }
    ]
  }, function (err, data) {
    if (err) {
      return res.status(500).json({
        success: false,
        message: '服务端错误'
      })
    }
 });
```
如果存在，就返回错误码

```javascript
    if (data) {
      // 邮箱或者昵称已存在
      return res.status(200).json({
        err_code: 1,
        message: 'Email or nickname aleady exists.'
      })
      return res.send(`邮箱或者密码已存在，请重试`)
    }
```
如果不存在，就进行保存，返回注册成功码，并重定向到主页面
<font  color=red>（注意：在服务端异步操作中是不能进行重定向的，具体解决方案可以参考我的另一篇文章[https://yangyongli.blog.csdn.net/article/details/114322377](https://yangyongli.blog.csdn.net/article/details/114322377)）
```javascript
    // 对密码进行 md5 重复加密
    body.password = md5(md5(body.password))

    new User(body).save(function (err, user) {
      if (err) {
        return res.status(500).json({
          err_code: 500,
          message: 'Internal error.'
        })
      }

      // 注册成功，使用 Session 记录用户的登陆状态
      req.session.user = user

      // Express 提供了一个响应方法：json
      // 该方法接收一个对象作为参数，它会自动帮你把对象转为字符串再发送给浏览器
      res.status(200).json({
        err_code: 0,
        message: 'OK'
      })
```

## 3. 完整代码
### app.js

```javascript
var express = require('express')
var path = require('path')
var bodyParser = require('body-parser')
var session = require('express-session')
var router = require('./router')

var app = express();

app.use('/public/',express.static(path.join(__dirname,'./public')));
app.use('/node_modules/',express.static(path.join(__dirname,'./node_modules')));

// 模板引擎 art-template
app.engine('html',require('express-art-template'))
app.set('views',path.join(__dirname,'./views/'));

// 配置解析表单 POST 请求体插件（注意：一定要在 app.use(router) 之前 ）
// parse application/x-www-form-urlencoded
app.use(bodyParser.urlencoded({ extended: false }))
// parse application/json
app.use(bodyParser.json())

app.use(session({
  // 配置加密字符串，它会在原有加密基础之上和这个字符串拼起来去加密
  // 目的是为了增加安全性，防止客户端恶意伪造
  secret: 'itcast',
  resave: false,
  saveUninitialized: false // 无论你是否使用 Session ，我都默认直接给你分配一把钥匙
}))

// 把路由挂载到 app 中
app.use(router);

app.listen(3000,function(){
    console.log('server is running.....');
})

```
### router.js

```javascript
var express = require('express');
// 数据库模块
var User = require('./models/user');    
var md5 = require('blueimp-md5');

var router = express.Router();

router.get('/',function(req,res){
    res.render('index.html',{
        user : req.session.user
    })
});


router.get('/login',function(req,res){
    res.render('login.html')
})

router.post('/login',function(req,res){
    var body= req.body;

    User.findOne({
        email: body.email,
        password: md5(md5(body.password))
    },function(err,user){
        if (err) {
             return res.status(500).json({
               err_code: 500,
               message: err.message
             })
         }
         // 如果邮箱和密码匹配，则 user 是查询到的用户对象，否则就是 null
        if(!user){
            return res.status(500).json({
                err_code : 1,
                message : '用户名或密码不正确 .'
            });
        }

        // 用户存在 登陆成功 通过 session 记录登陆状态
        req.session.user = user;

        res.status(200).json({
            err_code: 0,
            message: '登陆成功'
        });

    });
})



router.get('/register',function(req,res){
    res.render('register.html');
});

router.post('/register',function(req,res){
    // 1.获取表单数据
    var body = req.body;
    console.log("************************1************************* ");
    console.log(body);
    // 2.操作数据库
      //    判断改用户是否存在
      //    如果已存在，不允许注册
      //    如果不存在，注册新建用户
    User.findOne({
        $or: [
            {
                email:body.email
            },
            {
                nickname:body.nickname
            }
        ]
    },function(err,data){
        // 服务器错误
        console.log("************************2************************* ");
        if(err){
            return res.status(500).json({
                success: false,
                message: '服务端错误'
            });
        }

        // 判断改用户是否存在
        if (data) {
          // 邮箱或者昵称已存在
          return res.status(200).json({
            err_code: 1,
            message: 'Email or nickname aleady exists.'
          })
          return res.send(`邮箱或者昵称已存在，请重试`)
        }

        // 用户不存在 新建用户
        // 对密码进行 md5 加密
        body.password = md5(md5(body.password));

        new User(body).save(function(err,user){
            if(err){
                return res.status(500).json({
                  err_code: 500,
                  message: '注册失败.'
                })
            }
            // 注册成功，使用 Session 记录用户的登陆状态
            req.session.user = user;

            // 3.发送相应
            res.status(200).json({
              err_code: 0,
              message: 'OK'
            })

        });
    });  
    


});



// 个人信息 修改
router.get('/settings/profile', function (req, res) {
  res.render('./settings/profile.html',{
    user: req.session.user
  })
})

// 用户密码管理
router.get('/settings/admin',function(req,res){
  res.render('./settings/admin.html',{
    user: req.session.user
  })
})


router.get('/logout', function (req, res) {
  // 清除登陆状态
  req.session.user = null

  // 重定向到登录页
  res.redirect('/login')
})


module.exports = router;


```
### user.js

```javascript
var mongoose = require('mongoose')

// 连接数据库
mongoose.connect('mongodb://localhost/test', { useMongoClient: true })

var Schema = mongoose.Schema

var userSchema = new Schema({
  email: {
    type: String,
    required: true
  },
  nickname: {
    type: String,
    required: true
  },
  password: {
    type: String,
    required: true
  },
  created_time: {
    type: Date,
    // 注意：这里不要写 Date.now() 因为会即刻调用
    // 这里直接给了一个方法：Date.now
    // 当你去 new Model 的时候，如果你没有传递 create_time ，则 mongoose 就会调用 default 指定的Date.now 方法，使用其返回值作为默认值
    default: Date.now
  },
  last_modified_time: {
    type: Date,
    default: Date.now
  },
  avatar: {
    type: String,
    default: '/public/img/avatar-default.png'
  },
  bio: {
    type: String,
    default: ''
  },
  gender: {
    type: Number,
    enum: [-1, 0, 1],
    default: -1
  },
  birthday: {
    type: Date
  },
  status: {
    type: Number,
    // 0 没有权限限制
    // 1 不可以评论
    // 2 不可以登录
    enum: [0, 1, 2],
    default: 0
  }
})

module.exports = mongoose.model('User', userSchema)

```
### register.html


```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>用户注册</title>
  <link rel="stylesheet" href="/node_modules/bootstrap/dist/css/bootstrap.css">
  <link rel="stylesheet" href="/public/css/login.css">
</head>

<body>
  <div class="main">
    <div class="header">
      <a href="/">
        <img src="/public/img/logo3.png" alt="" width="100px">
      </a>
      <h1>用户注册</h1>
    </div>
    <!-- 
      表单具有默认的提交行为，默认是同步的，同步表单提交，浏览器会锁死（转圈儿）等待服务端的响应结果。
      表单的同步提交之后，无论服务端响应的是什么，都会直接把响应的结果覆盖掉当前页面。

      后来有人想到了一种办法，来解决这个问题。
     -->
    <form id="register_form" method="post" action="/register">
      <div class="form-group">
        <label for="email">邮箱</label>
        <input type="email" class="form-control" id="email" name="email" placeholder="Email" autofocus>
      </div>
      <div class="form-group">
        <label for="nickname">昵称</label>
        <input type="text" class="form-control" id="nickname" name="nickname" placeholder="Nickname">
      </div>
      <div class="form-group">
        <label for="password">密码</label>
        <input type="password" class="form-control" id="password" name="password" placeholder="Password">
      </div>
      <button type="submit" class="btn btn-success btn-block">注册</button>
    </form>
    <div class="message">
      <p>已有账号? <a href="/login">点击登录</a>.</p>
    </div>
  </div>
  <script src="/node_modules/jquery/dist/jquery.js"></script>
  <script>
    $('#register_form').on('submit', function (e) {
      e.preventDefault()
      var formData = $(this).serialize()
      $.ajax({
        url: '/register',
        type: 'post',
        data: formData,
        dataType: 'json',
        success: function (data) {
          var err_code = data.err_code
          if (err_code === 0) {
            // window.alert('注册成功！')
            // 服务端重定向针对异步请求无效
            window.location.href = '/'
          } else if (err_code === 1) {
            window.alert('邮箱已存在！')
          } else if (err_code === 2) {
            window.alert('昵称已存在！')
          } else if (err_code === 500) {
            window.alert('服务器忙，请稍后重试！')
          }
        }
      })
    })
  </script>
</body>

</html>

```
### login.html
```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>用户登录</title>
  <link rel="stylesheet" href="/node_modules/bootstrap/dist/css/bootstrap.css">
  <link rel="stylesheet" href="/public/css/login.css">
</head>

<body>
  <div class="main">
    <div class="header">
      <a href="/">
        <img src="/public/img/logo3.png" alt="" width="100px">
      </a>
      <h1>用户登录</h1>
    </div>
    <form id="login_form">
      <div class="form-group">
        <label for="">邮箱</label>
        <input type="email" class="form-control" id="" name="email" placeholder="Email" autofocus>
      </div>
      <div class="form-group">
        <label for="">密码</label>
        <a class="pull-right" href="">忘记密码？</a>
        <input type="password" class="form-control" id="" name="password" placeholder="Password">
      </div>
      <div class="checkbox">
        <label>
          <input type="checkbox">记住我
        </label>
      </div>
      <button type="submit" class="btn btn-success btn-block">登录</button>
    </form>
    <div class="message">
      <p>没有账号? <a href="/register">点击创建</a>.</p>
    </div>
  </div>
  <script src="/node_modules/jquery/dist/jquery.js"></script>
  <script>
    $('#login_form').on('submit', function (e) {
      e.preventDefault()
      var formData = $(this).serialize()
      console.log(formData)
      $.ajax({
        url: '/login',
        type: 'post',
        data: formData,
        dataType: 'json',
        success: function (data) {
          var err_code = data.err_code
          if (err_code === 0) {
            // window.alert('注册成功！')
            // 服务端重定向针对异步请求无效
            window.location.href = '/'
          } else if (err_code === 1) {
            window.alert('邮箱或者密码错误')
          } else if (err_code === 500) {
            window.alert('服务器忙，请稍后重试！')
          }
        }
      })
    })
  </script>
</body>

</html>

```
### 各个模板
home.html

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <title>{{block 'title'}}默认标题{{/block}}</title>
  <link rel="stylesheet" href="/node_modules/bootstrap/dist/css/bootstrap.css">
  {{block 'head'}}{{/block}}
</head>

<body>
  {{include '../_partials/header.html'}}
  {{block 'body'}}{{/block}}
  {{include '../_partials/footer.html'}}
  <script src="/node_modules/jquery/dist/jquery.js"></script>
  <script src="/node_modules/bootstrap/dist/js/bootstrap.js"></script>
  {{block 'script'}}{{/block}}
</body>

</html>

```
footer.html
```html
<footer class="container">
  <p>Copyright © 2017</p>
</footer>

```
header.html

```html
<nav class="navbar navbar-default">
  <div class="container">
    <!-- Brand and toggle get grouped for better mobile display -->
    <div class="navbar-header">
      <button type="button" class="navbar-toggle collapsed" data-toggle="collapse" data-target="#bs-example-navbar-collapse-1" aria-expanded="false">
        <span class="sr-only">Toggle navigation</span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
        <span class="icon-bar"></span>
      </button>
      <a class="navbar-brand" href="/">
        <img width="90px" src="/public/img/logo3.png" alt="">
      </a>
    </div>
    <!-- Collect the nav links, forms, and other content for toggling -->
    <div class="collapse navbar-collapse" id="bs-example-navbar-collapse-1">
      <form class="navbar-form navbar-left">
        <div class="form-group">
          <input type="text" class="form-control" placeholder="Search">
        </div>
      </form>
      <ul class="nav navbar-nav navbar-right">
        {{ if user }}
        <a class="btn btn-default navbar-btn" href="/topics/new">发起</a>
        <li class="dropdown">
          <a href="#" class="dropdown-toggle" data-toggle="dropdown" role="button" aria-haspopup="true" aria-expanded="false"><img width="20" height="20" src="../public/img/avatar-max-img.png" alt=""> <span class="caret"></span></a>
          <ul class="dropdown-menu">
            <li class="dropdown-current-user">
              当前登录用户: {{ user.nickname }}
            </li>
            <li role="separator" class="divider"></li>
            <li><a href="#">个人主页</a></li>
            <li><a href="/settings/profile">设置</a></li>
            <li><a href="/logout">退出</a></li>
          </ul>
        </li>
        {{ else }}
        <a class="btn btn-primary navbar-btn" href="/login">登录</a>
        <a class="btn btn-success navbar-btn" href="/register">注册</a>
        {{ /if }}
      </ul>
    </div>
    <!-- /.navbar-collapse -->
  </div>
  <!-- /.container-fluid -->
</nav>

```
setting-nav.html

```html
<div class="col-md-3 list-group">
  <a href="#" class="list-group-item disabled">用户设置</a>
  <a href="/settings/profile" class="list-group-item active">基本信息</a>
  <a href="/settings/admin" class="list-group-item">账户设置</a>
</div>

```
admin.html

```html
{{extend '../_layouts/home.html'}}

{{block 'title'}}{{'多人博客 - 首页'}}{{/block}}

{{block 'body'}}
<section class="container">
  {{include '../_partials/settings-nav.html'}}
  <div class="col-md-5">
    <div class="subhead">
      <h2>修改密码</h2>
    </div>
    <hr>
    <form>
      <div class="form-group">
        <label for="exampleInputPassword1">当前密码</label>
        <input type="password" class="form-control" id="exampleInputPassword1" placeholder="">
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">新的密码</label>
        <input type="password" class="form-control" id="exampleInputPassword1" placeholder="">
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">确认密码</label>
        <input type="password" class="form-control" id="exampleInputPassword1" placeholder="">
      </div>
      <button type="submit" class="btn btn-success">保存</button>
    </form>
    <div class="subhead">
      <h2>注销账号</h2>
      <hr>
    </div>
    <div>
      <p>一旦注销，不可找回，请谨慎操作</p>
      <button class="btn btn-danger">确认注销</button>
    </div>
    <hr>
  </div>
</section>
{{/block}}

```
profile.html

```html
{{extend '../_layouts/home.html'}}

{{block 'title'}}{{'多人博客 - 首页'}}{{/block}}

{{block 'body'}}
<section class="container">
  {{include '../_partials/settings-nav.html'}}
  <div class="col-md-5">
    <form>
      <div class="form-group">
        <label for="exampleInputEmail1">账号</label>
        <p class="form-control-static">{{ user.email }}</p>
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">昵称</label>
        <input type="input" class="form-control" id="exampleInputPassword1" placeholder="" value="{{ user.nickname }}">
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">介绍</label>
        <textarea class="form-control" rows="3"></textarea>
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">性别</label>
        <div>
          {{ if user.gender==0 }}
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio1" value="option1"> 男
          </label>
          <label class="radio-inline">
            <input checked="true" type="radio" name="inlineRadioOptions" id="inlineRadio2" value="option2"> 女
          </label>
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio3" value="option3"> 保密
          </label>
          {{ else if user.gender==1 }}
          <label class="radio-inline">
            <input checked="true" type="radio" name="inlineRadioOptions" id="inlineRadio1" value="option1"> 男
          </label>
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio2" value="option2"> 女
          </label>
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio3" value="option3"> 保密
          </label>
          {{ else }}
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio1" value="option1"> 男
          </label>
          <label class="radio-inline">
            <input type="radio" name="inlineRadioOptions" id="inlineRadio2" value="option2"> 女
          </label>
          <label class="radio-inline">
            <input checked="true" type="radio" name="inlineRadioOptions" id="inlineRadio3" value="option3"> 保密
          </label>
          {{ /if }}
        </div>
      </div>
      <div class="form-group">
        <label for="exampleInputPassword1">生日</label>
        <input type="password" class="form-control" id="exampleInputPassword1" placeholder="">
      </div>
      <button type="submit" class="btn btn-success">保存</button>
    </form>
  </div>
  <div class="col-md-2 profile-avatar">
    <dl>
      <dt>头像设置</dt>
      <dd>
        <img class="avatar" width="150" height="150" src="{{ user.avatar }}" alt="">
        <div>
          <button class="btn btn-default" href="">上传新头像</button>
        </div>
      </dd>
    </dl>
  </div>
</section>
{{/block}}

```
