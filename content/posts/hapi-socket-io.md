---
title: "Using hapi.js with Socket.io"
date: 2015-03-03T00:00:00+01:00
---
Socket.io and hapi.js are two great pieces of software for Node. There's no official documentation on how they work together though. I've seen questions about this several times on Twitter and Github, so I thought I'd write a quick tutorial to show just how easy it is to integrate the two.

### The listener

Every hapi server comes with a `listener` property:

    var Hapi = require('hapi');

    var server = new Hapi.Server();
    server.connection({ port: 4000 });

    var listener = server.listener; // <--- The listener

That `listener` object, in this case, is a node `http` (or `https`) server. Exactly the same as you would get from the following:

    var Http = require('http');

    var listener = Http.createServer(function (req, res){
    	...
    });

Whenever you create a new connection in hapi, it goes and creates a new `http` server for you internally. This gets set to `this.listener`.

### Setting up Socket.io

The offical Socket.io docs show how to set up socket.io when using a vanilla node `http` server.

	var handler = function (req, res) {
      ...
    };

    var app = require('http').createServer(handler);
    var io = require('socket.io')(app); // <--- what's this 'app' thing?

    app.listen(80);

The `app` variable in the above example is an instance of a node `http` server, just like our hapi server's `listener`.

We can take this knowledge that socket.io can be initialized with an `http` server over to hapi:

    var Hapi = require('hapi');

    var server = new Hapi.Server();
    server.connection({ port: 4000 });

    var io = require('socket.io')(server.listener);

    io.on('connection', function (socket) {

		socket.emit('Oh hii!');

        socket.on('burp', function () {
        	socket.emit('Excuse you!');
        });
	});

    server.start();

Pretty straightforward right? Socket.io will go and attach itself to the listener (server). You can make sure it's working by going to http://localhost:4000/socket.io/socket.io.js, you should see the socket.io client JS file being served.

### Why isn't there some kind of conflict between hapi and socket.io?

Every Node `http` server can have several listener (yeah...a different meaning of listener) functions for the `request` event. When you setup socket.io, it will remove all existing listeners (including the one added by hapi):

    var listeners = server.listeners('request').slice(0);
    server.removeAllListeners('request');

By removing all existing listeners, socket.io is ensuring it gets to the request first.

But it will keep hold of any others and if it decides the request isn't intended for socket.io (i.e. the url doesn't start /socket.io), it will call those other listener function instead, giving control back to hapi.

### How about when using multiple connections?

A "server" in hapi is semantically different to a "server" in node. A server in hapi may be listening on several different ports all at once by using multiple connections.

If your hapi server has multiple connections, it really has multiple node `http` servers running for you in the background. The `listener` property of the hapi server is always going to point to the `http` server on the first connection. If you want to attach socket.io to a different connection, you can use labels and `server.select()`:

	var Hapi = require('hapi');

    var server = new Hapi.Server();
    server.connection({ port: 4000, labels: ['api'] });
    server.connection({ port: 4001, labels: ['chat'] });

    var io = require('socket.io')(server.select('chat').listener);

    io.on('connection', function (socket) {

		socket.emit('Oh hii!');

        socket.on('burp', function () {
        	socket.emit('Excuse you!');
        });
	});

    server.start();

### What about plugins?

My favourite feature of hapi is plugins without a doubt. I love the way you can split up your concerns into separate modules. If I'm using socket.io with hapi, I always embrace the plugin approach to split my realtime concerns into their own plugin.

    .
    ├── chat	<--- My hapi plugin
    │   ├── handlers.js
    │   └── index.js
    └── index.js

##### index.js

    var Hapi = require('hapi');

    var server = new Hapi.Server();

    server.connection({ port: 4000, labels: ['api'] });
    server.connection({ port: 4001, labels: ['chat'] });

    server.register(require('./chat'), function (err) {

        if (err) {
            throw err;
        }

        server.start();
    });

##### chat/index.js

    var Handlers = require('./handlers');

    exports.register = function (server, options, next) {

        var io = require('socket.io')(server.select('chat').listener);

        io.on('connection', function (socket) {

            console.log('New connection!');

            socket.on('hello', Handlers.hello);
            socket.on('newMessage', Handlers.newMessage);
            socket.on('goodbye', Handlers.goodbye);
        });

        next();
    };

    exports.register.attributes = {
        name: 'hapi-chat'
    };

##### chat/handlers.js (socket.io is kind enough to bind your handlers to the socket so `this` is equal to the socket object)

    exports.hello = function () {

        this.emit('Hi back at you');
    };

    exports.newMessage = function (newMessage) {

        console.log('Got message', newMessage);
    };

    exports.goodbye = function () {

        this.emit('Take it easy, pal');
    };

I find this is a really clean approach to divide up apps that use both hapi and socket.io.

### That's it!

I hope this short tutorial has been of some help you to. If you've any suggestestions or corrections, please leave a comment or get me at @mt_harrison.

### The book

<div style="height: 200px">

<img style="float:left; overflow:hidden; padding: 0px 20px 20px 0px" src='http://matt-harrison.com/content/images/2015/Feb/hjsia_cover.jpg'/>I'm currently in the process of writing a book about hapi. <a href="http://manning.com/harrison/">Hapi.js in Action</a> is available now as a MEAP from Manning Publications. Use my author code <strong>mlharrison</strong> to get 50% off, for a limited time only.

</div>
