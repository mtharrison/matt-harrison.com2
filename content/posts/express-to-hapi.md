---
title: "Express to Hapi.js"
date: 2014-07-23T00:00:00+01:00
---
This isn't an X is better than Y post. I love Express, I still think it's a really great module and I've used it successfully in many projects.

That being said, I'm hearing good things about Hapi.js (referred to as Hapi from hereon in) recently which is another HTTP server framework for Node.js. So I figured it was time to check it out. As most people who I imagine come to Hapi, I have experience with Express and I'm wondering how it differs. This post is an in-depth examination of the 2 frameworks; their offerings, similarities and differences. It will split over 2 posts. This is **part 1**.

_Note: Part 2 has been temporarily delayed whilst I work on a Hapi Book (Hapi.js in Action for Manning). I'll post a discount code for my readers once the book is released._

Part 1 will look at:

- Making a server
- Adding routes
- Inspecting requests
- Working with responses

Part 2 will look at:

- Rendering views
- Routers
- Middleware and plugins
- Packs in Hapi.js
- Conclusion

## What is Hapi?

Hapi was developed by Eran Hammer and the team at Walmart labs, who found that Express wasn't meeting their needs for maintainability and extensibility.

Eran wrote a [great post on his blog](http://hueniverse.com/2012/12/20/hapi-a-prologue/) that you should read. There, Eran sums up the main motivation for creating Hapi:

> hapi was created around the idea that configuration is better than code, that business logic must be isolated from the transport layer, and that native node constructs like buffers and stream should be supported as first class objects. But most importantly, it was created to provide a modern, comprehensive environment in which as much of the effort is spent delivering business value.


## Making a server

Arguably the simplest thing to do in either framework is creating a server that binds and listens on a port.

**Only using Node's http module**

	var http = require('http');
	var server = http.createServer(function(){});

	server.listen(3000, function(){
		console.log("Listening on port 3000");
	});

**Express**

	var express = require('express');
	var app = express();

	var server = app.listen(3000, function(){
		console.log('Listening on port %d', server.address().port);
	});

**Hapi**

	var Hapi = require('hapi');
	var server = new Hapi.Server(3000);

	server.start(function () {
	    console.log('Server running at:', server.info.uri);
	});

Nothing too different here except Hapi takes a port number as you initialize a server rather than at the listen stage. Personally, I prefer this subtly different approach because it reduces the distance in code between creation and configuration of the server.

The reason I've also shown this example using only the `http` library is because I want to highlight a difference between Express and Hapi. Node's `http.createServer` takes a callback as an argument and it is this callback that will be evaluated with each HTTP request to the server. The call to `express()` returns a function (named `app` above) which is a callback which can be passed to `http.createServer`. So essentially your whole Express app is a callback for node's built in http server. In fact, Express' `listen` method is just a wrapper around both of Node `http`'s `createServer` and `listen`.

From express source (lib/application.js):

	app.listen = function(){
	  var server = http.createServer(this);
	  return server.listen.apply(server, arguments);
	};

In contrast, Hapi's constructor provides a instance of a Hapi server which is essentially its own universe and is mapiulated using its API.

I find Express's one-to-one mapping to the built in node module here quite natural and elegant.

## Adding routes

Without routes your server isn't very interesting. This is where the 2 frameworks start to diverge a little more.

Here's an example of creating a route which responds to "GET /" and returns "Hello, World!":

**Express**

	app.get('/', function(req, res){
	  res.send('Hello, World!');
	});

Here we can see express' strong influence from Sinatra, using a similar API as the popular Ruby library:

	get '/' do
	  'Hello world!'
	end

**Hapi**

	server.route({
	    method: 'GET',
	    path: '/',
	    handler: function (request, reply) {
	        reply('Hello, world!');
	    }
	});

A few things to notice that are different in Hapi:

- `server.route` takes a single argument which is a configuration object

- The HTTP verb you're responding to doesn't dictate the API method to use unlike express (`app.get()`, `app.post()` etc), it's always `server.route` and the HTTP verb is an option given as a string. If you want to respond with this handler to any HTTP verb you can specify the method as `"*"`. For similar functionality on Express, you'd use the `app.all()` method
- The response parameter in the handler function (`reply`) is a function which you call with the body you wish to send

If you have the need to dynamically generate routes from some external config, I feel Hapi would be cleaner here. For a good example of this being done, check out the [hapi-ninja](https://github.com/poeticninja/hapi-ninja) example on Github.

### Inspecting requests

In all but the most simple route handlers will need to do some examination of the HTTP request. Whether that's looking at request headers, decoding the body into structured data (as with form data or JSON) or reading cookies.

**Express**

In express, every route handler receives 3 arguments:

	app.get('/', function(req, res, next){
		res.send('Hello, World!');
	});

`req` is the request object within which all the meaningful HTTP request data is parsed into. Some of the most useful properties/methods are:

- `req.headers` - The requests headers
- `req.params` - Named parameters in the URL path
- `req.query` - Named parameters in the URL query string
- `req.param()` - Convenient sugar to get a parameter from any of the above

There's a lot of other useful stuff and I would refer you to the [documentation](http://expressjs.com/api#req.params) or even better the [code](https://github.com/visionmedia/express/blob/master/lib/request.js), rather than repeating it all here. Express' request object is a decorated, extended version of Node's [http.IncomingMessage](http://nodejs.org/api/http.html#http_http_incomingmessage) object as can be seen here (express/lib/request.js):

	var req = exports = module.exports = {
	  __proto__: http.IncomingMessage.prototype
	};

**Hapi**

Hapi provides much the same data as Express within its `request` object with some extra sugar for authentcation and [domains](http://nodejs.org/api/domain.html).

Hapi provides access to the node request object through a property `request.raw`. Hapi defines a [request lifecycle](http://hapijs.com/api#request-lifecycle) with predefined 'hooks' or extension points that plugins can 'hook into' e.g. `onRequest`, `onPreAuth` and `onPostAuth`.

There are also a set of `request` events which one can use to package up orthogonal behaviour. Here's an example (taken from the Hapi docs) that creates an SHA1 hash of all each request body:

	var Crypto = require('crypto');
	var Hapi = require('hapi');
	var server = new Hapi.Server();

	server.ext('onRequest', function (request, reply) {

	    var hash = Crypto.createHash('sha1');
	    request.on('peek', function (chunk) {

	        hash.update(chunk);
	    });

	    request.once('finish', function () {

	        console.log(hash.digest('hex'));
	    });

	    request.once('disconnect', function () {

	        console.error('request aborted');
	    });
	});

For more info about Hapi's request object, [read the docs](http://hapijs.com/api#request-object).

## Working with responses

**Express**

Express wraps the [`http.ServerResponse`](http://nodejs.org/api/http.html#http_class_http_serverresponse) object, which offers basic methods like `setHeader()`, `write()` and `end()`.

Express adds a bunch of really useful _chainable_ convenience methods, allowing you to write things like:

	res.set({
	  'Content-Type': 'image/png',
	  'Content-Length': '123',
	  'ETag': '12345'
	})
	.status(404)
	.sendfile('/path/to/404.png');

`res.send` in Express can take various data types (`string`, `Buffer`, `object`) and will set the appropriate `content-type` and convert the data to a `Buffer` for writing into the response stream.

Express also has a [`res.render()`](http://expressjs.com/api#res.render) method. In my opinion this is one of the most useful Express features if you're making a website. `res.render()` takes the path of a `view` file which can be `html` or a template written in a template language, such as [jade](http://jade-lang.com/) or [ejs](http://embeddedjs.com/) and a set of local variables to be passed to the view. The rendered HTML is then written into the response. An optional callback parameter will receive the rendered HTML as a parameter in case you need to do further processing or inspect the HTML before before writing it out.

**Hapi**

In Hapi, the second parameter in a handler function is not the 'response' object, but rather an object called `reply`.

When `reply()` is called as a function, it will return the `response` object and continue executing through the rest of the handler function, and in `process.nextTick()`, it will send the response.

This means after calling `reply()` you can still modify the response until the handler function returns.

Like Express, you can pass `reply()` a variety of data types and it will figure out what it needs to send.

Hapi's [response object](http://hapijs.com/api#response-object) manages things like setting headers, etags and performing redirects much in the same way Express does.

[Discuss on Hacker News](https://news.ycombinator.com/item?id=8089315)
