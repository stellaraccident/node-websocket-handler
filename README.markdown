node-websocket-handler
======================

There are a few different WebSocket related servers for node.js.  The most mature/used is Socket.IO.  It provides
a multi-protocol abstraction for a fair number of transports in an attempt to get the most compatibility.  In
addition it is fully operated "outside" of the server request handling mechanism.  If you want multi-protocol
abstraction, want something that has some history to it and is proven, and you are ok using something that integrates outside of the rest of your node.js http framework, then I highly recommend Socket.IO.

node-websocket-handler takes a different approach.  It is just WebSockets.  It currently supports draft76 (as
implemented by current versions of Chrome/Safari et al) as well as a stab at implementation of draft03 (which
is actually more recent than draft76), which is hopefully one of the last major breaking changes in the spec
before it is finalized.

In addition, this library makes an attempt to integrate with the native http server request processing paradigm
versus standing next to it.  At the main server level, you need to call a method that configures the server (this
actually is registering an "upgrade" event handler and taking action on it) and then wherever you are processing
your requests, you need to look for an "res.webSocket" property on endpoints that you expect to be websockets.  If
you want to handle it as a websocket request, just register some event listerners on res.webSocket and call
res.webSocket.connect().  Otherwise, return a response (typically an error) as usual.  I took this approach
because it lets WebSocket servers play as part of my http application instead of beside it.  Authorization, cookie
handling etc, stays as-is and then on the endpoints that need it, I can handle the WebSocket.

Example With Vanilla Node Http Server
-------------------------------------
This is the simplest example.  You can run it from demo/stdDemo.js.  It just sets up an echo server.

	var http=require('http');
	var webSocketHandler=require('../lib/webSocketHandler');

	function handleSocket(s) {
		s.on('message', function(body) {
			console.log('Got message: "' + body.toString('utf8') + '"');
			s.sendTextMessage(body);
		});
	
		s.connect();
	}

	var server=http.createServer(function(req, res) {
		var ws=res.webSocket;
		if (ws) {
			handleSocket(ws);
		} else {
			// Punt - we don't do anything else here
			res.writeHead(500);
			res.end('We only speak WebSockets here');
		}
	});

	webSocketHandler.configureServer(server);

	server.listen(4080);
	console.log('Server running');

Example Integrating with a Connect-based App
--------------------------------------------
This example sets up a boiler-plate Connect-based app and handles one of the endpoints as a WebSocket echo server.  You can run it from demo/connectDemo.js.  If you run it, open your WebSocket enabled browser to http://localhost:4080/echotest.html to see a simple page that demos it.

	var connect=require('connect'),
		webSocketHandler=require('../lib/webSocketHandler');

	function echoServer(ws) {
		ws.on('message', function(body) {
			console.log('Got message: "' + body.toString('utf8') + '"');
			ws.sendTextMessage(body);
		});
	
		// Take ownership of the request
		ws.connect();
	}


	var router=connect.router(function(app) {
		app.get('/echoserver', function(req, res, next) {
			var ws=res.webSocket;
			if (ws) {
				echoServer(ws);
			} else {
				// Punt - we don't do anything else here
				res.writeHead(500);
				res.end('We only speak WebSockets here');
			}
		});
	});

	var server=connect.createServer(
		connect.favicon(),
		connect.logger(),
		connect.staticProvider(__dirname + '/public'),
		router
	);

	webSocketHandler.configureServer(server);

	server.listen(4080);
	console.log('Server running');

Integration with other approaches/frameworks should be similar.  The key things to note are:

	* You must call `webSocketHandler.configureServer(server)` to register the upgrade handler that makes everything work
	* When you handle a request that has a `webSocket` instance on its response, you can take special action
	* In order to actually use the WebSocket, you should add events to the `res.webSocket` and then call `res.webSocket.connect()`.  Because of the way the protocol is framed and may come over the wire, it is important to add event listeners before calling connect.  There may already be data pending and your listeners may fire immediately.
	* Once connected you can call `res.webSocket.sendTextMessage(buffer_or_string)` or `res.webSocket.sendBinaryMessage(buffer_or_string)` (only on new protocol)

Status
------
I originally mocked this library up to test an Android WebSocket client I was writing (https://github.com/stellaeof/java-websocket-client).  I'm still evolving both and they are just now minimally functional.  I'll be adding real tests soon.

Both the draft76 and draft03 protocols are implemented but I tested the draft03 protocol against a client that I wrote and it may have problems.  It should be considered a work in progress and I intend to have it track to the latest spec.

The following are wholly unimplemented at present:
	* Fragment handling
	* Extensions
	* Initiating a ping (although everything exists to respond to it and emit events if one is received)

These parts add complexity to the draft03 protocol for little benefit (in my opinion) and I am waiting to see how they fare in the final spec.

License
=======
Feel free to do whatever you want with it.  It's under the MIT license.

The MIT License

Copyright (c) 2011 Stella Laurenzo

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.

