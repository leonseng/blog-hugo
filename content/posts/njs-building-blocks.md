---
title: "NGINX Javascript building blocks"
date: 2022-01-05T16:21:56+11:00
tags: ['nginx', 'njs']
categories: ['tech']
---

[NGINX Javascript (njs)](https://nginx.org/en/docs/njs/) provides a powerful and modern extension to configure NGINX. Looking at the list of [use cases](https://www.nginx.com/blog/harnessing-power-convenience-of-javascript-for-each-request-with-nginx-javascript-module/#Use-Cases-for-the-NGINX-JavaScript-Module) left me bedazzled, but also very confused on how njs was used to achieve those use cases. So here another post of me learning about the building blocks provided by njs. Hopefully we will have a better frame of mind to understand how the njs is utilized in those use cases by the end of this post.

All the source code referenced in this post can be found here:

> [https://github.com/leonseng/nginx-everything/tree/master/nginx-javascript](https://github.com/leonseng/nginx-everything/tree/master/nginx-javascript)


# Setup

The njs module package needs to be installed per instructions [here](https://nginx.org/en/docs/njs/install.html). Alternatively, just use the [nginx Docker image](https://hub.docker.com/_/nginx) which has the njs package included, albeit slightly outdated (v0.6.2 as of the day this post was written).

To use njs in your NGINX configuration, load the module and specify where the njs files are with the [js_path](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_path) and [js_import](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_import) directives:
```nginx
# nginx.conf
load_module modules/ngx_http_js_module.so;

http {
  js_path "/etc/nginx/njs/";
  js_import main from main.js;
}
```


> njs actually provides two modules [ngx_http_js_module](https://nginx.org/en/docs/http/ngx_http_js_module.html) and [ngx_stream_js_module](https://nginx.org/en/docs/stream/ngx_stream_js_module.html), but I'll be focusing on HTTP traffic in this post.

The example above assumes there's an njs file in `/etc/nginx/njs/main.js`. All exported functions in `main.js` can be referenced within `nginx.conf` using the `<module>.<function>` format, e.g. `main.foo`. [js_import](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_import) can be called multiple times if there are more njs files.

Now, we are ready to look into some of the building blocks that njs provides.

# Building Blocks

## Building a response body in JS

The [js_content](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_content) directive enables you to build a response body in njs. Here's an example of an njs function returning a simple string
```nginx
# nginx.conf
http {
  server {
    location /hello {
      js_content main.hello;
    }
  }
}
```

```javascript
// main.js
function hello(r) {
  r.return(200, "Hello world\n")
}
```

The function can be updated to return `JSON` data if `Accept: application/json` is provided in the request header
```javascript
function hello(r) {
  r.return(
    200,
    (
      ("Accept" in r.headersIn && r.headersIn["Accept"] == "application/json") ?
        JSON.stringify({ "body": "Hello world" }) :
        "Hello world\n"
    )
  )
  return
}
```

```cmd
$ curl localhost/hello
Hello world

$ curl -H "Accept: application/json" localhost/hello
{"body":"Hello world"}
```


## Logging in njs

Logging in njs is provided by the following functions:
- [r.log()](https://nginx.org/en/docs/njs/reference.html#r_log) - on `info` level
- [r.error()](https://nginx.org/en/docs/njs/reference.html#r_error) - on `error` level
- [r.warn()](https://nginx.org/en/docs/njs/reference.html#r_warn) - on `warning` level

Here's an example where we use all three logging functions in our `hello` function:

```javascript
function hello(r) {
  r.log("My info log")
  r.warn("My warning log")
  r.error("My error log")
  ...
}
```

When the `hello` function is invoked, notice that only the string passed to [r.error()](https://nginx.org/en/docs/njs/reference.html#r_error) shows up in `stderr`:
```console
$ curl localhost/hello
Hello world

$ docker logs njs
...
2022/01/04 05:39:00 [error] 32#32: *1 js: My error log
...
```

The reason we don't see the entries from [r.log()](https://nginx.org/en/docs/njs/reference.html#r_log) and [r.warn()](https://nginx.org/en/docs/njs/reference.html#r_warn) is because the njs logging functions write to the **error log**, and by default, NGINX only logs `error` level entries to the error log, as defined by the [error_log](http://nginx.org/en/docs/ngx_core_module.html#error_log) directive. This means that only [r.error()](https://nginx.org/en/docs/njs/reference.html#r_error) can be used for njs logging unless the [error_log](http://nginx.org/en/docs/ngx_core_module.html#error_log) directive is modified in `nginx.conf`.


## Making subrequests

njs supports making additional requests while handling an existing request using the [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest). This can be handy for situations such as getting NGINX to perform HTTP requests or API calls to multiple endpoints on the client's behalf, and building a response based on the responses from each of the subrequests, simplifying the client side logic.

Here's an example where NGINX performs a subrequest to an external endpoint [worldtimeapi.org](http://worldtimeapi.org), parses the JSON response from the subrequest to retrieve the value of the `datetime` field, and returns it to the client.
```nginx
# nginx.conf
http {
  server {
    location /time {
      js_content main.time;
    }

    # only required for r.subrequest()
    location /proxy/worldtimeapi/ {
      internal;
      proxy_pass http://worldtimeapi.org/;
      proxy_set_header Host worldtimeapi.org;
    }
  }
}
```

```javascript
// main.js
/*
  Perform a subrequest to World Time API and return the datetime field as a response
*/
function time(r) {
  r.subrequest(
    '/proxy/worldtimeapi/api/timezone/Australia/Melbourne',
    { method: "GET" },
    function (res) {
      let subreq_response = JSON.parse(res.responseBuffer)
      r.return(res.status, subreq_response.datetime)
    }
  )
}
```

```console
$ curl localhost/time
2022-01-04T16:48:32.025819+11:00
```

A couple things to note:
1. You may have noticed an additional internal location `/proxy/worldtimeapi` referenced in `nginx.conf`. This is because [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) only supports sending requests to the NGINX reverse proxy itself. Hence, we have to define the internal location which proxies requests from [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) to the external destination `worldtimeapi.org`.

    > Performing a subrequest to an external endpoint directly will fail
    > ```javascript
    > // main.js
    > function time(r) {
    >   r.subrequest(
    >     'http://worldtimeapi.org/api/timezone/Australia/Melbourne',
    >     { method: "GET" },
    >     function (res) {
    >       let subreq_response = JSON.parse(res.responseBuffer)
    >       r.return(res.status, subreq_response.datetime)
    >     })
    > }
    > ```
    >
    > You'll end up with an error like this
    > ```log
    > 2022/01/05 06:03:35 [error] 31#31: *1 open() "/etc/nginx/htmlhttp://worldtimeapi.org/api/timezone/Australia/> Melbourne" failed (2: No such file or directory), client: 172.18.0.1, server: , request: "GET /time HTTP/1.1", > subrequest: "http://worldtimeapi.org/api/timezone/Australia/Melbourne", host: "localhost"
    > ```

1. The example uses javascript callback to handle the response to the subrequest. njs 0.7 now supports the `async` and `await` pattern, which makes the code more readable:
    ```javascript
    async function time(r) {
      let res = await r.subrequest('/proxy/worldtimeapi/api/timezone/Australia/Melbourne')
      let subreq_response = JSON.parse(res.responseBuffer)
      r.return(res.status, json.datetime)
    }
    ```

    > To check the version of njs, run
    > ```cmd
    > # njs -v
    > 0.6.2
    > ```

## Making subrequests - part 2

There is another function [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch) that allows you to make sideband calls:
```javascript
// main.js
function time(r) {
  ngx.fetch('http://worldtimeapi.org/api/timezone/Australia/Melbourne')
    .then(res => res.json())
    .then(body => r.return(200, body.datetime))
    .catch(e => r.return(501, e.message));
```

> Note that [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch) allows you to directly target an external endpoint without going through an NGINX location.

So how do we choose between [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) vs [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch)? Here are a couple of key differences:

1. [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) **MUST** target the reverse proxy, i.e. an NGINX location. Whilst it seems like more boilerplate code to reach an external endpoint (via [proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)), it also means you can configure features like caching ([proxy_cache](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache)) or mutual TLS ([proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key)/[proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate)) for the subrequest.

    [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch) can target any endpoint within njs itself, but it comes at a cost detailed below.

1. [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) doesn't need to create a new HTTP connection for the subrequest, hence is more efficient.

    [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch) on the other hand is designed for making HTTP requests on behalf on `stream` traffic which don't have an existing HTTP connection, meaning it needs to create a new HTTP connection for the request.

1. [r.subrequest()](https://nginx.org/en/docs/njs/reference.html#r_subrequest) shares its input/request headers with the client request (but can be modified with the [proxy_set_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive), whereas [ngx.fetch()](http://nginx.org/en/docs/njs/reference.html#ngx_fetch) doesn't.




## Modifying response

Responses from upstream can be modified using njs via the [js_header_filter](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_header_filter) and [js_body_filter](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_body_filter) directives.

In the example below, we proxy a call to [httpbin.org](https://httpbin.org) via the `/httpbin/` location, and modify the response by
1. adding a HTTP header `X-njs`, and
1. adding a new field `njs` to the JSON data in the response body

before sending it back to the client.
```nginx
# nginx.conf
http {
  server {
    location /httpbin/ {
      js_header_filter main.modify_response_header;
      js_body_filter main.modify_response_body;

      proxy_set_header Host httpbin.org;
      proxy_pass http://httpbin.org/;
    }
  }
}
```

```javascript
// main.js
function modify_response_body(r, data) {
  if (data !== "") {
    let body = JSON.parse(data)
    body["njs"] = "modify_response_body";
    r.sendBuffer(JSON.stringify(body));
  } else {
    r.sendBuffer(data);
  }

  return
}

function modify_response_header(r) {
  r.headersOut["X-njs"] = "modify_response_header";
  delete r.headersOut["Content-Length"];
  return
}
```

> Note that in `main.modify_response_header`, the `Content-Length` header is deleted to enforce chunked transfer encoding.
>
> This is required as the length of the response body changed when the new field `njs` was added in `main.modify_response_body`, and I couldn't figure out how to retrieve the new content-length in the `main.modify_response_header` function. `js_header_filter` is always called before `js_body_filter`, regardless of how they are ordered in the `nginx.conf`.
>
> Another one for the **"too hard, maybe someday"** pile.

We can verify on the client that the new header `X-njs` and new field `njs` are added in the response:
```console
$ curl -s -vvv localhost/njs-res/headers | jq .
*   Trying 127.0.0.1:80...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /njs-res/headers HTTP/1.1
> Host: localhost
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.3
< Date: Wed, 05 Jan 2022 03:28:35 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
< Access-Control-Allow-Origin: *
< Access-Control-Allow-Credentials: true
< X-njs: modify_response_header
<
{ [172 bytes data]
* Connection #0 to host localhost left intact
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.68.0",
    "X-Amzn-Trace-Id": "Root=1-61d51060-52324c5109e28ffe77c10dcb"
  },
  "njs": "modify_response_body"
}
```


## Pass me that variable

[js_var](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_var) declares a writable variable, meaning its value can be set and modified by njs. This is handy when you need to pass the output from one directive to the next for the same request.

The example below emulates an authentication flow (via the [auth_request](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html#auth_request) directive) which returns a session token and stores it in `$session_token`. The `$session_token` variable is then used as a value to a HTTP header `X-session-token` via a [proxy_set_header](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) directive, and sent to the remote destination [httpbin.org](httpbin.org).

```nginx
# nginx.conf
http {
  js_var $session_token "0000";
  ...

  server {
    location /httpbin/ {
      auth_request /auth;

      proxy_set_header "X-session-token" $session_token;
      proxy_set_header Host httpbin.org;
      proxy_pass http://httpbin.org/;
    }

    location /auth {
      internal;
      js_content main.auth;
    }
  }
}
```

```javascript
// main.js
/*
  Retrieves a UUID from httpbin.org/uuid,
  stores it in session_token js_var, and
  returns 200 to simulate a successful auth flow
*/
function auth(r) {
  ngx.fetch('http://httpbin.org/uuid')
    .then(reply => reply.text())
    .then(body => {
      r.variables.session_token = JSON.parse(body).uuid;
      r.error("Auth successful with session token: " + r.variables.session_token);
      r.return(200);
    })
    .catch(e => r.return(501, e.message));
}
```

We can verify this by sending a request to `/httpbin/headers`, which echoes back the HTTP headers in the request. The log entry shows the session token returned from the auth request in `main.auth`, and the curl response shows the same value in the `X-Session-Token` header
```console
$ curl -s localhost/httpbin/headers | jq .
{
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "User-Agent": "curl/7.68.0",
    "X-Amzn-Trace-Id": "Root=1-61d67585-01adbb996ae4a8ae005d1c38",
    "X-Session-Token": "0b101558-66aa-4dbf-b6cd-307e905d9364"
  }
}

$ docker logs njs
...
2022/01/06 04:53:28 [error] 31#31: *1 js: Auth successful with session token: 0b101558-66aa-4dbf-b6cd-307e905d9364
```


## Complex variable evaluation

njs can perform complex evaluation of variables in `nginx.conf` with the [js_set](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_set) directive.

In the example below, we use [js_set](https://nginx.org/en/docs/http/ngx_http_js_module.html#js_set) to declare a variable `$hashed_req_id`, which has its value derived from the njs function `main.hash_req_id` each time it is evaluated. In this case, `main.hash_req_id` is called each time a response body is formed for the location `/hash-req-id`:
```nginx
# nginx.conf
http {
  js_set $hashed_req_id main.hash_req_id;

  server {
    location /hash-req-id {
      return 200 $hashed_req_id;
    }
  }
}
```

```javascript
// main.js
var cr = require('crypto')

/*
  Returns the SHA1 hash of the request ID
 */
function hash_req_id(r) {
  return cr.createHash('sha1').update(r.variables.request_id).digest('base64url');
}
```

Making a HTTP call to `/hash-req-id` returns the content of the variable `hashed_req_id`
```console
$ curl -s localhost/hash-req-id
9-6jrW9xEBmwKP2lvi4Suba1bXk
```

Here's another example of the variable being referenced in the [log_format](http://nginx.org/en/docs/http/ngx_http_log_module.html#log_format) directive:
```nginx
# nginx.conf
http {
  log_format custom_log '$remote_addr [$time_local] hash_req_id returns $hashed_req_id';
  access_log /dev/stdout custom_log;
}
```

Requests to NGINX now produce log entries as such:
```log
172.18.0.1 [05/Jan/2022:04:18:55 +0000] hash_req_id returns 9-6jrW9xEBmwKP2lvi4Suba1bXk
```


# Closing

njs is a powerful toolbox to extend your NGINX configurations. Hopefully this post has given you some food for thought on some good ideas, otherwise, head over [here](https://www.nginx.com/blog/harnessing-power-convenience-of-javascript-for-each-request-with-nginx-javascript-module/#Use-Cases-for-the-NGINX-JavaScript-Module) to see how others are using njs.
