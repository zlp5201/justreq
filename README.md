# justreq(JR Server)

[![NPM version][npm-image]][npm-url] [![Downloads][downloads-image]][npm-url] [![License][license-image]][npm-url]

A caching proxy server for testing interface of HTTP or HTTPS. A never offline testing interface server. A mock server. It can help us to develop offline. It's especially useful for Front-End developers.

## Features
* Each request will be cached. It can make us be absorbed in develop whether online or offline. 
* Mock server. We can use other file to mock interface, such as JSON, TXT and so on.
* We make an script named JRS, just like PHP. It can mock more flexible interface, and even develop a site.
* Support for ES6, ES7, can develop more efficient.
* Support for CORS(Cross-Origin Resource Sharing), so it can work for Web Front-End.
* No invasive codes, do not inject any code to our project.

## Install
Install [Node.js](https://nodejs.org/en/) first, then run this to install CLI of JUSTREQ
```
npm install -g justreq-cli
```

Install main program using:
```
npm install justreq
```

## Initialization
To create a configuration file using:
```
justreq init
```
After finished, the file ".justreq" will be created at working directory. We can modify it any time.

## Usage
To start a JR Server using:
```
justreq start
```

Then modify our testing interface to JR Server, eg.
```javascript
// const API_HOST = "https://test.yourhost.com";
const API_HOST = "http://127.0.0.1:8000";
$.get(API_HOST + "/getInfo.do?userId=1001", callback);
```

******
 
Also, we can run this to clean caches and start JR server
```
justreq start -c
```

Or, run this to temporarily proxy other remote not in configuration file
```
justreq start -h temp.werhost.com
```

We can run this to get help
```
justreq start --help
```

## Advancement
### JR Script
Now, I will introduce our ***jrs***. It's Javascript-based, so we can use it without study any more. Let me show:
```javascript
// getUser.jrs
var userId = $_GET['userId'];
var users = {
    1001 : {name:'Bruce', age: 22},
    1002 : {name:'Lily', age: 21}
};
var user = users[userId];
setCookie('userName', user.name);
echo(JSON.stringify(user));
end();
```
We added some necessary global function and properties, except usual of JS.

#### Properties
|   name   |  description  
|----------|:-----------------------------------------------------
| $_GET    | An associative object of variables passed to the current script via the URL parameters.
| $_POST   | An associative object of variables passed to the current script via the HTTP POST method
| $_COOKIE | An associative object of variables passed to the current script via HTTP Cookies.
| $_HEADER | An associative object of variables passed to the current script via HTTP headers
| $_FILES  | An associative object of items uploaded to the current script via the HTTP POST method when using multipart/form-data as the HTTP Content-Type in the request.

#### Functions
|   name                 |  description  
|------------------------|:---------------------------------------
| echo(string)           | Output one string.
| end([string])          | To end the current script and output one string. ***Tips. we must always use this function to finish our script, or it will run until timeout.***
| sendFile(filepath)     | Output an file and end the script. Using this, we don't have to use "end()".
| setMime(suffix)        | Modify mimetype of current script. By default, try using "json", switch to "txt" when failure. Options: txt, html, css, xml, json, js, jpg, jpeg, gif, png, svg. Also, we can use "setHeader()" to customize "Content-Type".
| setCookie(name, value) | Send a cookie. usage: setCookie(name, value [, expires [, path [, domain [, secure [, httponly]]]]])
| setHeader(name, value) | Send a HTTP header. But, it can't be sent about "Server" and "Date", they are automatic by JR Server.

For ***jrs***, we can use any modules from Node.js, because it run in Node.js runtime environment. Such as [Faker.js](https://github.com/marak/Faker.js/), [mockjs](https://github.com/nuysoft/Mock) and so on.
And, we can code with ES6, ES7 also, if Node.js enough new.


*********

### Rules
For some special interface, we can rule it using:

|    name    |  description  
|------------|:-----------------------------------------------------
| href       | The path of URL, not null. Support for RegExp.
| ignoreArgs | Some fields can be ignored, such as version stamp "?v=1483884433384", we can using: `{"ignoreArgs" : "v"}`
| noCache    | Not allow caching. Default is allowed.
| subs       | Substitution's path. Suggest to use our ***jrs***, or "json", "txt" and so on.

Example:

```json
// .justreq
{
    ...
  "rules": [
    {
      "href":       "user.do\\?id=(\\d+)",
      "subs":       "user.jrs?userId=$1"
    },
    {
      "href":       "login.do",
      "noCache":    true
    },
    {
      "href":       "getGoodsInfo.do",
      "ignoreArgs": "v,token,timestamp"
    }
  ]
}
```

*********

### Inspector
Sometimes, we need another way to decide whether cache the client request or not. Such as follow case. The post data is encode with base64, but a field is insignificant. If we do not intervene to inspect, JUSTREQ will cache repeat.

#### Examples
```javascript
// myInsp.js
const querystring = require('querystring');
const crypto = require('crypto');
const base64 = require('base64-utf8');

function md5(str) {
  let md5sum = crypto.createHash('md5');
  md5sum.update(str);
  str = md5sum.digest('hex');
  return str;
}

function insp(req, buf) {
  let rawData = buf.toString('utf8'); // token=F2F0CF28&encrypt=eyJhcnRpY2xlSWQiOjk5LCJtaXN0IjoiWTJodiJ9
  let postData = querystring.parse(rawData); // {token:"F2F0CF28", encrypt:"eyJ..."}
  if (postData.encrypt) {
    let decodeString = base64.decode(postData.encrypt);
    let payload = JSON.parse(decodeString); // {"articleId":99,"mist":"Y2hv"}
    let md5Code = md5(req.method + req.url + payload.articleId);
    return {needCache: true, cacheId: md5Code}; // need to be cached
  } else {
    return null; // inspector should skip this request
  }
}

module.exports = insp; // Must be exported it as node module
```
And add a configurition to ".justreq"
```json
{
  ...
  "inspector": ".jr/myInsp.js"
}
```

#### Standard for "insp.js"
```javascript
/**
 * @param  {object} req The req create by client request
 * @param  {buffer} buf The data from FormData
 * @return {json}       {needCache: <boolean>, cacheId: <md5>} or null
 */
function insp(req, buf) {
  ...
  return {needCache: <boolean>, cacheId: <md5>};
}
module.exports = insp; // Must be exported it as node module
```
##### Notice:
* Expect the value of "cacheId" by return as an md5 code. Recommend `md5(req.method + req.url + bufData)`, to avoid conflict from cache.
* To skip inspect, we can just return null.
* To avoid http choke, we can't use any asynchronous code and `setTimeout`, `setInterval`.


*********

### Configurations
|    name        |  description  
|----------------|:-----------------------------------------------------
| host           | Not null, the hostname of remote interface server.
| port           | Optional, the port of remote interface server, default is 80. (Connet interface with HTTPS, if it's 443)
| cacheTime      | Optional, the time of caches update. value in "h", "m", "s". Default is "20m".
| cachePath      | Optional, cache files directory, default is ".jr/cache".
| substitutePath | Optional, substitution files directory, default is ".jr/subs".
| jrPort         | Optional, the port of JR server, default is 8000.
| proxyTimeout   | Optional, the timeout of proxy, value in "h", "m", "s", default is "6s".
| proxyHttps     | Optional, whether interface server is running on HTTPS or not. Options: "auto", "yes", "no". Default is "auto".
| ssl_ca         | Optional, CA(Certificate Authority) path, if interface server is running on HTTPS and need it.
| ssl_key        | Optional, the path of private key to use for SSL, if interface server is running on HTTPS and need it.
| ssl_cert       | Optional, the path of public x509 certificate to use, if interface server is running on HTTPS and need it.
| onCors         | Optional, on CORS(Cross-Origin Resource Sharing)? Options: "yes", "no". Default is "yes".
| inspector      | Optional, custom respector script for decide whether cache request or not.<br>Expect the return of script as `{needCache: <boolean>, cacheId: <md5>}`
| rules          | Optional, refer to [RULES](#user-content-rules)

*********

## DEMO
There are some demo in "./examples" directory.

Open the directory and open `run_examples.cmd`(*shell `./run_examples` for Linux*) to start JUSTREQ. Also, we can start JUSTREQ using `justreq start`.

Now, we can open any html files to experience.

[jrs.html](examples/jrs.html), [substitutes.html](examples/substitutes.html), [upload.html](examples/upload.html)

*********

## ChangeLog
### 2017-3-2
#### v0.3.4
* modify "readme.md"

#### v0.3.3
* fix bug about run JR Server without inspector failed

### 2017-3-1
#### v0.3.2
* add inspector to inspect http request customized
* fix wrong reponse when read cache when failed proxy

### 2017-2-28
#### v0.3.1
* refactor some files use ES6
* fix bug about create new slow map fail
* optimize proxy pipe
* add some global property to JRScript
* add 404 err to JR Server when can't find jrs file
* fix wrong charaters which create by reduceFormData.js when length below 17

### 2017-2-20
#### v0.2.13
* Removed character '^M' in some files

### 2017-2-9
#### v0.2.12
* Fixed a exception that when submit a form which contain some checkbox

### 2017-2-4
#### v0.2.11
* Support to add 'search' in subs of RULE

#### v0.2.10
* Support for RegExp replacement

### 2017-1-19
#### v0.2.9
* Fixed a exception about proxy timeout

### 2017-1-17
#### v0.2.8
* Optimized cleaning cache logic

#### v0.2.7
* Optimized processing logic of CORS
* Normalized some code

### 2017-1-15
#### v0.2.6
* Optimized log printing for substitutions
* Modified some error description in README.md

### 2017-1-13
#### v0.2.5
* Added english "README.md" and examples

#### v0.2.4
* Optimized log printing
* Removed unusual dependencies

### 2017-1-12
#### v0.2.3
* Fixed an exception about read substitution file

#### v0.2.2
* Added a demo about upload file of jrs
* Packet less fixed

### 2017-1-11
#### v0.2.1
* Refactored "server.js" using modules of HTTP and "pipe"
* Refactored "proxy.js" using "pipe"
* Refactored "jrs.js" using "formidable"
* Support for HTTPS using HTTPS module
* Added error page of HTTP status 400
* Added configuration of CORS

### 2017-1-9
#### v0.1.3
* A buf of set-cookie fixes.

### 2017-1-8
#### v0.1.2
* Support for CORS
* Finished examples

## More
[justreq](https://github.com/vilien/justreq)  - github

[issue](https://github.com/vilien/justreq/issues)

[Chinese](https://github.com/vilien/justreq/blob/master/README-cn.md)

[blog[CN]](http://blog.csdn.net/binjly/article/details/54238070)

## License
Released under the [![MIT license][license-image]](https://github.com/vilien/justreq/blob/master/MIT-LICENSE)

[downloads-image]: https://img.shields.io/npm/dm/justreq.svg
[license-image]: https://img.shields.io/npm/l/justreq.svg?style=flat
[npm-image]: https://img.shields.io/npm/v/justreq.svg
[npm-url]: https://www.npmjs.com/package/justreq