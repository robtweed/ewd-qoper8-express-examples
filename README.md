# ewd-qoper8-express-examples

Rob Tweed <rtweed@mgateway.com>  
16 August 2017, M/Gateway Developments Ltd [http://www.mgateway.com](http://www.mgateway.com)  

Twitter: [@rtweed](https://twitter.com/rtweed)

Google Group for discussions, support, advice etc: [http://groups.google.co.uk/group/enterprise-web-developer-community](http://groups.google.co.uk/group/enterprise-web-developer-community)

© 2017 M/Gateway Developments Ltd

---

# Table of Contents

  * [Integrating ewd-qoper8 with Express](#toe-integrating-ewd-qoper8-with-express)
  * [ewd-qoper-express](#toe-ewd-qoper-express) 
  * [Sending Error Responses from your Worker Module](#toe-sending-error-responses-from-your-worker-module) 
  * [Express Router](#toe-express-router)
  * [Using WebSockets With ewd-qoper8 and Express](#toe-using-websockets-with-ewd-qoper8-and-express)


## <a id="toe-integrating-ewd-qoper8-with-express"></a>Integrating ewd-qoper8 with Express

ewd-qoper8 is not really designed to be a standalone module. More sensibly, it is designed to be used in conjunction with a server module of some sort. The very popular and increasingly standard web server module Express is a good example of something with which ewd-qoper8 can be easily integrated.

So, for example, ewd-qoper8's worker processes can be used to handle incoming HTTP requests, with the output from your worker-process module message handlers being
returned to the browser or web server client.

Here's a simple example of ewd-qoper8 working with Express. You'll find this in the /examples directory - look for express1.js:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();
app.post('/qoper8', function (req, res) {
  q.handleMessage(req.body, function (response) {
    res.send(response);
  });
});

q.on('started', function () {

  this.worker.module = process.cwd() + '/examples/modules/express-module1';

  var server = app.listen(8080, function () {

    var host = server.address().address
    var port = server.address().port

    console.log("EWD-Express listening at http://%s:%s", host, port);
    console.log('__dirname = ' + __dirname);
  });
});

q.start();

```
This example will start ewd-qoper8, and when it has started and ready, it will start Express, listening on port 8080. It also sets the worker module file to be the express-module.js file
that you'll also find in the /examples/modules folder.

The example worker module is pretty simple, just echoing back the incoming message and adding a few properties of its own:
```javascript
'use strict';

module.exports = function () {

  this.on('message', function (messageObj, send, finished) {
    
    var results = {
      youSent: messageObj,
      workerSent: 'hello from worker ' + process.pid,
      time: new Date().toString()
    };
    finished(results);
  });

};

```

Express will be watching for incoming HTTP POST requests with a URL path of /qoper8. The body of these requests will contain a JSON payload. Note the use of the standard bodyParser.json middleware to parse the JSON and make it available as req.body.
This JSON payload is then handled by ewd-qoper8's standard `handleMessage()` function.

This queues the message and awaits the response from the ewd-qoper8 worker process to which the message was dispatched. The returned result object is provided to you in its callback function which then uses Express's `res.send()` method to return it to the client.

Try using a REST client (eg the Chrome Advanced REST Client extension), and POST to the URL /qoper8 a JSON payload such as this (Make sure the Content-Type is set to application/json):
```json
{
 "type": "HTTPMessage",
 "a": "hello"
}
```
You should get a response back such as this:
```json
{
  "type": "HTTPMessage",
  "finished": true,
  "message": {
    "youSent": {
      "type": "HTTPMessage",
      "a": "hello"
    },
    "workerSent": "hello from worker 13809Э",
    "time": "Fri Feb 19 2016 02:02:33 GMT+0000 (GMT)"
  }
}
```


## <a id="toe-ewd-qoper-express"></a>ewd-qoper-express

To make integration with Express even easier and more powerful, you can install a module named [ewd-qoper8-express](https://github.com/robtweed/ewd-qoper8-express):

```
npm install ewd-qoper8-express
```

You'll find an example of how to use this in /examples directory: look for the file express2.js. You'll see that it contains the following:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');
var qx = require('ewd-qoper8-express');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();
qx.addTo(q);

app.post('/qoper8', function (req, res) {
  qx.handleMessage(req, res);
});

app.get('/qoper8/test', function (req, res) {
  qx.handleMessage(req, res);
});

q.on('started', function () {
  this.worker.module = process.cwd() + '/examples/modules/express-module1';
  var server = app.listen(8080);
});
q.start();

```

Note the way you add ewd-qoper8-express to ewd-qoper8's master process:
```javascript
var qx = require('ewd-qoper8-express');
var q = new qoper8.masterProcess();
qx.addTo(q);
```

ewd-qoper8-express provides us with a special version of the `handleMessage()` method which constructs a message from all the relevant parts of the incoming Express request object and automatically returns the response received from the worker that processed it:
```
qx.handleMessage(req, res);
```

This example is using the same worker module as before, so it should just echo back the request generated by ewd-qoper8-express's handleMessage method.

Start up the express2.js example and try POST'ing the same request as before. You should get a response back such as this:
```json
{
  "youSent": {
    "type": "ewd-qoper8-express",
    "path": "/qoper8",
    "method": "POST",
    "headers": {
      "host": "192.168.1.188:8080",
      "connection": "keep-alive",
      "content-length": "44",
      "user-agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/48.0.2564.109 Safari/537.36",
      "origin": "chrome-extension://hgmloofddffdnphfgcellkdfbfbjeloo",
      "authorization": "f0906e12-10de-4cfc-a5e6-eaf4430d6764",
      "content-type": "application/json",
      "accept": "*/*",
      "accept-encoding": "gzip, deflate",
      "accept-language": "en-US,en;q=0.8",
      "cookie": "state-666E1C02-315E-11E5-8917-0C29C5382300=SYSADM%3A0%2C0; Username=UnknownUser"
    },
    "params": {},
    "query": {},
    "body": {
      "type": "HTTPMessage",
      "a": "hello"
    }
  },
  "workerSent": "hello from worker 13684",
  "time": "Wed Mar 02 2016 06:33:35 GMT+0000 (GMT)"
}
```
You can see at once that the message that was constructed from Express's req object is much more complex than before (see the youSent part of the message, shown in bold):
- **type**: this is set to 'ewd-qoper8-express'
- **method**: matches the HTTP method
- **headers**: contains the HTTP request headers (req.headers)
- **params**: contains any req.params properties
- **path**: the URL path that was specified in the request
- **query**: contains any req.query properties (ie any URL name/value pairs)
- **body**: for POST and PUT requests, the parsed JSON payload

This ensures that your worker module will have all the information it needs to process the incoming Express requests.

ewd-qoper8-express also provides a special worker-side method to make things simpler for you.

You add this to your worker using the following:
```javascript
var handleExpressMessage = require('ewd-qoper8-express').workerMessage;
```
Take a look at the example worker process named express-module2.js in the /examples/modules directory:
```javascript
'use strict';

module.exports = function () {

  var handleExpressMessage = require('ewd-qoper8-express').workerMessage;

  this.on('expressMessage', function (messageObj, send, finished) {
    var results = {
      youSent: messageObj,
      workerSent: 'hello from worker ' + process.pid,
      time: new Date().toString()
    };
    finished(results);
  });

  this.on('message', function (messageObj, send, finished) {
    var expressMessage = handleExpressMessage.call(this, messageObj, send, finished);
    
    if (expressMessage) return;
    
    // handle any non-Express messages
    if (messageObj.type === 'non-express-message') {
      var results = {
        messageType: 'non-express',
        workerSent: 'hello from worker ' + process.pid,
        time: new Date().toString()
      };
      finished(results);
    } else {
      this.emit('unknownMessage', messageObj, send, finished);
    }
  });

};
```

This loads the ewd-qoper8-express worker method and applies it within the `on('message')` handler:

```javascript
var expressMessage = handleExpressMessage.call(this, messageObj, send, finished);
```

What this does is to filter out all incoming messages with a type of 'ewd-qoper8-express' and generates a new event named 'expressMessage'. You can see the handler for this message in the above example:
```javascript
 this.on('expressMessage', function (messageObj, send, finished) {..}
```
The reason for doing this is that you might also be sending non-Express-formatted messages from the ewd-qoper8 master process, and you want to keep their processing logic clearly separated.

This can be achieved by testing the return value from the `handleExpressMessage()` function. It will be true if the message was an ewd-qoper8-express type, and false if not. You can see how this is done in the example above.

You can try out this example - look for the file express3.js in the examples directory:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');
var qx = require('ewd-qoper8-express');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();
qx.addTo(q);

app.post('/qoper8', function (req, res) {
  qx.handleMessage(req, res);
});

app.get('/qoper8/test', function (req, res) {
  var request = {
    type: 'non-express-message',
    hello: 'world'
  };
  
  q.handleMessage(request, function (response) {
    res.send(response);
  });
});

app.get('/qoper8/fail', function (req, res) {
  var request = {
    type: 'unhandled-message',
  };

  q.handleMessage(request, function (response) {
    var message = response.message;
    
    if (message.error) {
      res.status(400).send(message);
    } else {
      res.send(message);
    }
  });
});

q.on('started', function () {
  this.worker.module = process.cwd() + '/examples/modules/express-module2';
  var server = app.listen(8080);
});

q.start();

```

POSTing a /qoper8 URL will send an ewd-qoper8-express formatted message to the worker module (express-module2).

However, sending /qoper8/test as a GET request will generate a manually-constructed message with a type of 'non-express-message' which will be handled as differently within the worker module. Sending /qoper8/fail as a GET request will return a "No handler found' error response.


## <a id="toe-sending-error-responses-from-your-worker-module"></a>Sending Error Responses from your Worker Module

Whether or not you're handling the special ewd-qoper8-express messages within your worker module, if you've added the ewd-qoper8-express module to your master process, you can very simply generate error responses that will be treated as such by Express.

In your worker module message handlers, simply structure and send your error responses as follows:
```javascript
var response = {
  error: 'Some error message'
};
finished(response);
```

Express will send out an error response with an HTTP status code of 400 with the JSON payload:
```json
{"error": "Some error message"}
```

If you want Express to use a different status code, simply define it in the response as in this example that generates a status code of 500:
```javascript
var response = {
  error: 'Something went wrong on the server!',
  status: {code: 500}
};
finished(response);
```

If you want to generate a standard ewd-qoper8 "No handler was found for this message" error response, you can simply do the following in your message handler. Take a look at express4.js in the /examples directory:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');
var qx = require('ewd-qoper8-express');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();
qx.addTo(q);

app.get('/qoper8/pass', function (req, res) {
  qx.handleMessage(req, res);
});

app.get('/qoper8/fail', function (req, res) {
  qx.handleMessage(req, res);
});

app.get('/qoper8/nohandler', function (req, res) {
  qx.handleMessage(req, res);
});
q.on('started', function () {
  this.worker.module = process.cwd() + '/examples/module/express-module3';
  var server = app.listen(8080);
});

q.start();

```

It loads express-module3.js as its worker module:
```javascript
'use strict';

module.exports = function () {

  this.on('message', function (messageObj, send, finished) {
    var results;
    
    if (messageObj.path === '/qoper8/pass') {
      results = {
        youSent: messageObj,
        workerSent: 'hello from worker ' + process.pid,
        time: new Date().toString()
      };
      finished(results);
      return;
    }

    if (messageObj.path === '/qoper8/fail') {
      results = {
        error: 'An error occurred!',
        status: {code: 403}
      };
      finished(results);
      return;
    }

    this.emit('unknownMessage', messageObj, send, finished);
  });

};
```

Try the following GET requests:
- /qoper8/pass should echo back your original message
- /qoper8/fail should return a 403 error
- /qoper8/nohandler should return a 400 'No handler found' error


## <a id="toe-express-router"></a>Express Router

ewd-qoper8-express also provides a pre-packaged version of Express Router. Simply use it as follows:
```javascript
app.use('/vista', qx.router());
```
This incorporates the `qx.handleMessage` function behind the scenes, so provides all its capabilities to your specified routing.

Take a look at express4.js in ewd-qoper8's /test directory:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');
var qx = require('ewd-qoper8-express');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();
qx.addTo(q);

app.use('/qoper8', qx.router());
q.on('started', function () {
  this.worker.module = process.cwd() + '/examples/modules/express-module1';
  var server = app.listen(8080);
});

q.start();

```

This example uses the simple worker module express-module.js which just echoes back all the messages you send. You'll now see that any requests (GET, POST etc) that start with /qoper8 will be sent as ewd-qoper8-express formatted messages to your worker module.

So, for example, a GET request to /qoper8/testing/a/b?hello=world
will return a response similar to this:
```json
{
  "youSent": {
    "type": "ewd-qoper8-express",
    "path": "/qoper8/testing/a/b?hello=world",
    "method": "GET",
    "headers": {
      "host": "192.168.1.188:8080",
      "connection": "keep-alive",
      "user-agent": "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/48.0.2564.109 Safari/537.36",
      "authorization": "f0906e12-10de-4cfc-a5e6-eaf4430d6764",
      "accept": "*/*",
      "accept-encoding": "gzip, deflate, sdch",
      "accept-language": "en-US,en;q=0.8",
      "cookie": "state-666E1C02-315E-11E5-8917-0C29C5382300=SYSADM%3A0%2C0; Username=UnknownUser"
    },
    "params": {
      "0": "a/b",
      "type":"testing"
    },
    "query": {
      "hello": "world"
    },
    "body": {},
    "application": "qoper8",
    "expressType": "testing"
  },
 "workerSent": "hello from worker 13859",
 "time": "Wed Mar 02 2016 08:45:24 GMT+0000 (GMT)"
}
```
Notice that the path element following the /qoper8 root is available to you in your worker module message handler as `messageObj.params.type`. The /x/y elements are available in
`messageObj.params['0']`.

Also notice that the URL query string `?hello=world` is available as `messageObj.query.hello`.


## <a id="toe-using-websockets-with-ewd-qoper8-and-express"></a>Using WebSockets With ewd-qoper8 and Express

ewd-qoper8 will also work with web socket messages. See the express6.js in the /examples directory which contains the following:
```javascript
'use strict';

var express = require('express');
var bodyParser = require('body-parser');
var qoper8 = require('ewd-qoper8');

var app = express();
app.use(bodyParser.json());

var q = new qoper8.masterProcess();

app.get('/', function (req, res) {
  res.sendFile(__dirname + '/index.html');
});

q.on('started', function () {

  this.worker.module = process.cwd() + '/examples/modules/express-module1';
  
  var q = this;
  var server = app.listen(8080);
  var io = require('socket.io')(server);
  
  io.on('connection', function (socket) {
    socket.on('my-request', function (data) {
      q.handleMessage(data, function (resultObj) {
        delete resultObj.finished;
        socket.emit('my-response', resultObj);
      });
    });
  });

});

q.start();

```
For simplicity, this example uses the same worker module as before that just echoes back any messges it receives. This time it also loads the socket.io module. The socket.io event handler for incoming web socket messages makes use of the standard ewd-qoper8 `handleMessage()` function. The only difference is that instead of returning the message using: `res.send(resultObj);` it returns a WebSocket message as the response using:
```javascript
socket.emit('my-response', resultObj);
```

In order to test this example, you need to load an HTML file - there's one already created in the /examples folder (index.html). If you start the express6.js script and put the following URL into a browser: <http://192.168.1.193:8080/> (change the IP address as appropriate), it should load the index.html page. This loads the client-side socket.io library and otherwise just contains a button that contains the text "Click Me”:
```html
<html>
<head>
  <title id="ewd-qoper8 Demo"></title>
</head>
<body>
  <script src="/socket.io/socket.io.js"></script>
  <script>
    var socket = io.connect();
    socket.on('my-response', function (data) {
      console.log('ewd-qoper8 message received: ' + JSON.stringify(data, null, 2));
    });

    function sendMessage() {
      var message = {
        type: 'testSocketRequest',
        hello: 'world'
      };
      socket.emit('my-request', message);
    }
  </script>
  <div>
    <button onClick='sendMessage()'>Click me</button>
 </div>
 </body>
</html>
```

When the button is clicked it will send a web socket message to Express/socket.io, and will show the returned response web socket message in the browser's Javascript console. It should look something like this:
```json
ewd-qoper8 message received: {
  "type": "testSocketRequest",
  "message": {
    "youSent": {
      "type": "testSocketRequest",
      "hello": "world"
    },
    "workerSent": "hello from worker 13912",
    "time": "Wed Mar 02 2016 09:04:38 GMT+0000 (GMT)"
  }
}
```

One of the important differences between HTTP and WebSockets is that you can send multiple messages to a browser from the back-end. Take a look at express7.js in the /examples directory - the only difference between it and express6.js is that it uses a different worker module: express-module4.js:
```javascript
'use script';

module.exports = function () {

  this.on('message', function (messageObj, send, finished) {
    send({
      intermediate: 'message',
      text: 'With websockets you can send multiple messages to the browser!'
    });

    var results = {
      youSent: messageObj,
      workerSent: 'hello from worker ' + process.pid,
      time: new Date().toString()
    };
    finished(results);
  });

};
```
This time, every time the button is clicked in the index.html file, you should see two messages arriving in the browser's JavaScript console:
```json
ewd-qoper8 message received: {
  "type": "testSocketRequest",
  "message": {
    "intermediate": "message",
    "text": "With websockets you can send multiple messages to the browser!"
  }
}
ewd-qoper8 message received: {
  "type": "testSocketRequest",
  "message": {
    "youSent": {
      "type": "testSocketRequest",
      "hello": "world"
    },
    "workerSent": "hello from worker 13943",
    "time": "Wed Mar 02 2016 09:16:00 GMT+0000 (GMT)"
  }
}
```
Intermediate messages should be sent using the `send()` function to ensure that the master process doesn't return the child process to the available pool.
