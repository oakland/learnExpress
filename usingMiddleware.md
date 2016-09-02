*这篇文章是官网[Using middleware](https://expressjs.com/en/guide/using-middleware.html)的译文，翻译这篇文章对于理解中间件有很大的帮助。在文章的最后，我会将翻译后的心得，也就是对于五种中间件的最大的特点进行一个简单的总结和归纳，便于从概念上把握这五种中间件的使用。*

# Using middleware

Express 本质上是一个路由分发和中间件配置的网络框架，这个框架本身的功能极其简单：**express 应用的本质就是一系列中间件的调用。**

中间件本质上就是一个函数，这个函数可以获取到请求（req），响应（res），以及在应用的请求-响应循环中的下一个中间件函数。下一个中间件函数通常由名为 next 的变量进行标识。

中间件函数可以执行如下任务：

- 执行任何的代码 
- 修改请求和响应对象
- 结束请求-响应循环
- 调用堆栈中的下一个中间件函数

如果当前中间件函数没有结束请求-响应循环，那么它**必须**调用 next() 来将控制权传递给下一个中间件函数。否则，请求将会被挂起。

一个 Express 应用可以使用如下类型的中间件：

- 应用级别中间件(Application-level middleware)
- 路由级别中间件(Router-level)
- 错误处理中间件(Error-handling)
- 内置中间件(Built-in)
- 第三方中间件(Third-party)

你可以使用挂在路径（可选）来载入应用级别中间件或者路由级别中间件。你也可以一次性载入一系列中间件函数，这会在某个挂载点上创建一个次级堆栈(sub-stack)的中间件系统。

## 应用级别的中间件

可以通过使用 app.use() 或者 app.METHOD() 来将应用级别的中间件绑定到 app 实例上，METHOD 是由中间件函数处理的 HTTP 的请求方式，例如 GET, PUT, POST 等，不过应该是小写的形式。

下面的例子展示了一个没有挂在路径的中间件。每次 app 获取到一个请求的时候，这个函数都会执行。

	var express = require('express');
	var app = express();
	
	app.use(function(req, res, next) {
	  console.log('Time: ' + Date.now());
	  next();
	});
	
下面的例子展示了一个挂在到 /user/:id 路径上的中间件函数。任何在 /user/:id 路径上的 HTTP 请求都会导致中间件函数的触发。

	app.use('/user/:id', function(req, res, next) {
	  console.log('Request Type: ' + req.method);
	  next();
	});
	
下例则是在某个挂在点上载入一系列中间件函数的例子。它展示了一个中间件次级堆栈，这个次级堆栈会打印出请求 /user/:id 时任何类型的 HTTP 请求信息。

	app.use('/user/:id', function(req, res, next) {
	  console.log('Request Url: ' + req.originalUrl);
	  next();
	}, function(req, res, next) {
	  console.log('Request Type: ' + req.method);
	  next();
	});
	
路由处理器允许你为一个路径定义多个路由。下面的例子为 /user/:id 的 GET 请求定义了两个路由。第二个不会导致任何问题，但是永远不会被调用，因为第一个路由已经结束了请求-响应循环。

下面的例子展示了一个处理 /user/:id 路径的 GET 请求的中间件次级堆栈。

	app.get('/user/:id', function(req, res, next) {
	  console.log('ID: '+ req.param.id);
	  next();
	}, function(req, res, next) {
	  res.send('User Info');
	});
	
	// handler for the /user/:id path, which prints the user ID
	app.get('/user/:id', function (req, res, next) {
	  res.end(req.params.id);
	});
	
要想从路由中间件堆栈中跳过剩下所有的中间件函数，调用 call('route') 就可以将控制权交给下一个路由。注意：next('route') 仅仅在通过 app.use() 和 app.METHOD() 载入的中间件函数中生效。

下例展示了用于处理 /user/:id 的 GET 请求的中间件次级堆栈。

	app.get('/user/:id', function(req, res, next) {
	  // if the user ID is 0, skip to the next route
	  if(req.param.id = 1) next('route');
	  // otherwise pass the control to the next middleware function in this stack
	  else next();
	}, function(req, res, next) {
	  // render a regular page
	  res.render('regular');
	});
	
	// handle for the /user/:id page, which renders a special page
	app.get('/user/:id', function(req, res, next) {
	  res.render('special');
	});
	
## 路由级别的中间件

路由级别的中间件和应用级别的中间件工作方式完全相同，除了一点，就是 路由级别的中间件是绑定到 express.Router() 的实例上的。

	var route = express.Router();
	
通过 route.use() 和 route.METHOD() 来载入路由级别中间件。

下例代码通过使用路由级别的中间件，复制了上面所示应用级别中间件的中间件系统：

	var app = express();
	var router = express.Router();

	// a middleware function with no mount path. This code is executed for every request to the router
	router.use(function (req, res, next) {
	  console.log('Time:', Date.now());
	  next();
	});

	// a middleware sub-stack shows request info for any type of HTTP request to the /user/:id path
	router.use('/user/:id', function(req, res, next) {
	  console.log('Request URL:', req.originalUrl);
	  next();
	}, function (req, res, next) {
	  console.log('Request Type:', req.method);
	  next();
	});

	// a middleware sub-stack that handles GET requests to the /user/:id path
	router.get('/user/:id', function (req, res, next) {
	// if the user ID is 0, skip to the next router
	if (req.params.id == 0) next('route');
	// otherwise pass control to the next middleware function in this stack
	else next(); //
	}, function (req, res, next) {
	// render a regular page
	res.render('regular');
	});

	// handler for the /user/:id path, which renders a special page
	router.get('/user/:id', function (req, res, next) {
	  console.log(req.params.id);
	  res.render('special');
	});

	// mount the router on the app
	app.use('/', router);
	
## 错误处理中间件

**错误处理中间件永远会接收四个参数。你必须提供四个参数来明确这个中间件为错误处理中间件函数。即使你不会使用 next 对象，你也必须指定它来保持这个标识。否则，next 对象就会被当做普通的中间件而无法进行错误处理。**

定义错误处理中间件函数的方式和定义其他中间件函数的方式相同，唯一的区别就在于错误处理中间件拥有四个参数，而不是三个，具体而言，则是 (err, req, res, next)：

	app.use(function(err, req, res, next) {
	  console.error(err.stack);
	  res.status(500).send('Something broke!');
	});

关于错误处理中间件的详细信息，请参考[Error handling。](https://expressjs.com/en/guide/error-handling.html)

## 内置中间件

从 4.x 版本开始，Express 不再依赖 [Connect](https://github.com/senchalabs/connect)。除了 express.static 之外，所以之前内置在 Express 中的中间件函数现在都位于不同的模块中。请参阅[the list of middleware functions](https://github.com/senchalabs/connect#middleware)。

现在仅有的内置中间件函数就是 express.static。这个函数是基于 [server-static](https://github.com/expressjs/serve-static) 的，用于服务静态资源，如 HTML文件，图片等。

函数的写法如下：

	express.static(root, [options])
	
root 参数指定提供静态资源的根目录。

关于这个函数的 options 参数的信息及细节，请参阅[epxress static](https://expressjs.com/en/4x/api.html#express.static)。

下面是一个使用 express.static 中间件的例子，采用了 elaborate options 对象：

	var options = {	
	  dotfiles: 'ignore',
	  etag: false, 
	  extensions: ['html', 'html'],
	  index: false,
	  maxAge: '1d',
	  redirect: false,
	  setHeaders: function(req, res, next) {
	    res.set('x-timestamp', Date.now());
	  }
	}
	
	app.use(express.static('public', options));
	
一个 app 可以拥有不止一个静态资源的目录：

	app.use(exprss.static('public'));
	app.use(epxress.static('uploads'));
	app.use(express.static('files'));
	
关于 [server-static](https://github.com/expressjs/serve-static) 的更多细节以及选项，请查看 [server-static](https://github.com/expressjs/serve-static) 文档。

## 第三方中间件

使用第三方中间件为 Express 应用添加更多功能。

安装需要引入的函数的 node 模块，接着接着在你的 app 的应用级别或者路由级别载入这些中间件模块。下例展示了安装和引入 cookie-parsing 中间件的函数  cookie-parser。

	npm install cookie-parser
	
	var express = require('express');
	var app = express();
	var cookieParser = requeire()
	
	// load the cookie-parsin middle
	
Express 中部分常用第三方中间件函数清单，清参阅[Third-party](https://expressjs.com/en/resources/middleware.html)。