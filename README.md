# Automatic browser refresh
####Automatic browser refresh when save a file

Besides reloading the Node application when the source code changes, you can also speed up development for web applications. Instead of manually triggering the page refresh in the browser, we can automate this as well using tools such as livereload.

They work similarly to the ones presented before, because they watch for file changes in certain folders and trigger a browser refresh in this case (instead of a server restart). The refresh is done either by a script injected in the page or by a browser plugin.

Instead of showing you how to use livereload, this time we will create a similar tool ourselves with Node. It will do the following:

* Watch for file changes in a folder;
* Send a message to all connected clients using server-sent events; and
* Trigger the page reload.

First we should install the NPM dependencies needed for the project:

1. [express](http://npm.im/express) - for creating the sample web application
2. [watch](http://npm.im/watch) - to watch for file changes
3. [sendevent](https://www.npmjs.com/package/sendevent) - server-sent events, SSE (an alternative would have been websockets)
4. [uglify-js](http://npm.im/uglify-js) - for minifying the client-side JavaScript files
5. [ejs](http://npm.im/uglify-js) - view templates

Next we will create a simple Express server that renders a home view on the front page:

```javascript
"use strict";

var express = require('express');
var app = express();
var ejs = require('ejs');
var path = require('path');

var PORT = process.env.PORT || 1337;

var reloadify = require('./lib/reloadify');
reloadify(app, __dirname + '/views');

// view engine setup
app.engine('html', ejs.renderFile);
app.set('views', path.join(__dirname, 'views'));
app.set('view engine', 'html');

// serve an empty page that just loads the browserify bundle
app.get('/', function(req, res) {
  res.render('home');
});

app.listen(PORT);
console.log('server started on port %s', PORT);
```
Since we are using Express we will also create the browser-refresh tool as an Express middleware. The middleware will attach the SSE endpoint and will also create a view helper for the client script. The arguments for the middleware function will be the Express **app** and the folder to be monitored. Since we know that, we can already add the following lines before the view setup (inside server.js):

```javascript
var reloadify = require('./lib/reloadify');
reloadify(app, __dirname + '/views');
```
We are watching the /views folder for changes. And now for the middleware:
```javascript
var sendevent = require('sendevent');
var watch = require('watch');
var uglify = require('uglify-js');
var fs = require('fs');
var ENV = process.env.NODE_ENV || 'development';

// create && minify static JS code to be included in the page
var polyfill = fs.readFileSync(__dirname + '/assets/eventsource-polyfill.js', 'utf8');
var clientScript = fs.readFileSync(__dirname + '/assets/client-script.js', 'utf8');
var script = uglify.minify(polyfill + clientScript, { fromString: true }).code;

function reloadify(app, dir) {
  if (ENV !== 'development') {
    app.locals.watchScript = '';
    return;
  }

  // create a middlware that handles requests to `/eventstream`
  var events = sendevent('/eventstream');

  app.use(events);

  watch.watchTree(dir, function (f, curr, prev) {
    events.broadcast({ msg: 'reload' });
  });

  // assign the script to a local var so it's accessible in the view
  app.locals.watchScript = '<script>' + script + '</script>';
}

module.exports = reloadify;
```
As you might have noticed, if the environment isn't set to 'development' the middleware won't do anything. This means we won't have to remove it for production.

The frontend JS file is pretty simple, it will just listen to the SSE messages and reload the page when needed:
```javascript
(function() {
  function subscribe(url, callback) {
    var source = new window.EventSource(url);

    source.onmessage = function(e) {
      callback(e.data);
    };

    source.onerror = function(e) {
      if (source.readyState == window.EventSource.CLOSED) return;

      console.log('sse error', e);
    };

    return source.close.bind(source);
  };

  subscribe('/eventstream', function(data) {
    if (data && /reload/.test(data)) {
      window.location.reload();
    }
  });

}());
```
The eventsource-polyfill.js is [Remy Sharp's polyfill for SSE](https://github.com/remy/polyfills/blob/master/EventSource.js) . Last but not least, the only thing left to do is to include the frontend generated script into the page (**/views/home.html**) using the view helper:
```javascript
...
<%- watchScript %>
...
```
Now every time you make a change to the home.html page the browser will automatically reload the home page of the server for you
(**http://localhost:1337/**).
