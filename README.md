# express-throttle
Request throttling middleware for Express framework

## Installation

```bash
$ npm install express-throttle
```

## Implementation

The throttling is done using the canonical [token bucket](https://en.wikipedia.org/wiki/Token_bucket) algorithm, where tokens are "refilled" in a sliding window manner (as opposed to a fixed time interval). This means that if we set maximum rate of 10 requests / minute, a client will not be able to send 10 requests 0:59, and 10 more 1:01. However, if the client sends 10 requests at 0:30, he will be able to send a new request at 0:36 (as tokens are regenerated continuously 1 every 6 seconds).

## Examples

```js
var express = require("express");
var throttle = require("express-throttle");
  
var app = express();
  
// Throttle to 5 reqs/s
app.post("/search", throttle("5/s"), function(req, res, next) {
  // ...
});
```
Combine it with a burst capacity of 10, meaning that the client can make 10 requests at any rate. The capacity is "refilled" with the specified rate (in this case 5/s).
```js
app.post("/search", throttle({ "rate": "5/s", "burst": 10 }), function(req, res, next) {
  // ...
});
  
// Decimal values for rate values are allowed
app.post("/search", throttle({ "rate": "0.5/s", "burst": 5 }), function(req, res, next) {
  // ...
});
```
By default, throttling is done on a per ip-address basis (respecting the X-Forwarded-For header). This can be configured by providing a custom key-function:
```js
var options = {
  "rate": "5/s",
  "burst": 10,
  "key": function(req) {
    return req.session.username;
  }
};

app.post("/search", throttle(options), function(req, res, next) {
  // ...
});
```
The "cost" per request can also be customized, making it possible to, for example whitelist certain requests:
```js
var whitelist = ["ip-1", "ip-2", ...];
  
var options = {
  "rate": "5/s",
  "burst": 10,
  "cost": function(req) {
    var ip_address = req.connection.remoteAddress;
      
    if (whitelist.indexOf(ip_address) >= 0) {
      return 0;
    } else if (req.session.is_privileged_user) {
      return 0.5;
    } else {
      return 1;
    }
  }
};
  
app.post("/search", throttle(options), function(req, res, next) {
  // ...
});

var options = {
  "rate": "1/s",
  "burst": 10,
  "cost": 2.5 // fixed costs are also supported
};

app.post("/expensive", throttle(options), function(req, res, next) {
  // ...
});
```
Throttled requests will simply be responded with an empty 429 response. This can be overridden:
```js
var options = {
  "rate": "5/s",
  "burst": 10,
  "on_throttled": function(req, res) {
    // Possible course of actions:
    // 1) Add client ip address to a ban list
    // 2) Send back more information
    res.set("X-Rate-Limit-Limit", 5);
    res.status(503).send("System overloaded, try again at a later time.");
  }
};
```
Throttling can be applied across multiple processes. This requires an external storage mechanism which can be configured as follows:
```js
function ExternalStorage(connection_settings) {
  // ...
}
  
// These methods must be implemented
ExternalStorage.prototype.get = function(key, callback) {
  fetch(key, function(entry) {
    // First argument should be null if no errors occurred
    callback(null, entry);
  });
}
  
ExternalStorage.prototype.set = function(key, value, callback) {
  save(key, value, function(err) {
    // err should be null if no errors occurred
    callback(err); 
  });
}
  
var options = {
  "rate": "5/s",
  "store": new ExternalStorage()
}
  
app.post("/search", throttle(options), function(req, res, next) {
  // ...
});
```

## Options

`rate`: Determines the number of requests allowed within the specified time unit before subsequent requests get throttled. Must be specified according to the following format: *decimal/time-unit*

*decimal*: A non-negative decimal value. This value is internally stored as a float, but scientific notation is not supported (e.g 10e2).

*time-unit*: Any of the following: `s, sec, second, m, min, minute, h, hour, d, day`

`burst`: The number of requests that can be made at any rate. The burst quota is refilled with the specified `rate`.

`store`: Custom storage class. Must implement a `get` and `set` method with the following signatures:
```js
// callback in both methods must be called with an error (if any) as first argument

function get(key, callback) {
  fetch(key, function(err, value) {
    if (err) callback(err);
    else callback(null, value);
  });
}
function set(key, value, callback) {
  // value will be an object with the following structure:
  // { "tokens": Number, "accessed": Number }
  save(key, value, function(err) {
    callback(err);
  }
}
```
Defaults to an LRU cache with a maximum of 10000 entries.

`key`: Function used to identify clients. It will be called with an [express request object](http://expressjs.com/en/4x/api.html#req). Defaults to:
```js
function(req) {
	return req.headers["x-forwarded-for"] || req.connection.remoteAddress;
}
```

`cost`: Number or function used to calculate the cost for a request. It will be called with an [express request object](http://expressjs.com/en/4x/api.html#req). Defaults to 1.

`on_throttled`: A function called when the request is throttled. It will be called with an [express request object](http://expressjs.com/en/4x/api.html#req) and [express response object](http://expressjs.com/en/4x/api.html#res). Defaults to:
```js
function(req, res) {
	res.status(429).end();
};
```