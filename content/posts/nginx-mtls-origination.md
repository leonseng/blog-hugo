---
title: "NGINX: mTLS Origination"
date: 2022-03-29T21:46:13+11:00
draft: false
tags: ['nginx', 'mtls']
categories: ['tech']
---

---

Whilst testing some egress control capabilities of [NGINX Service Mesh](https://github.com/nginxinc/nginx-service-mesh) x [NGINX Ingress Controller](https://github.com/nginxinc/kubernetes-ingress) (more on them in the future 😉), I had to configure NGINX to perform mutual TLS origination towards an upstream test server. The official NGINX documentation has a write-up on [Securing HTTP Traffic to Upstream Servers](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/), which shows how this can be done with the [proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate) and [proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key) directives:

```nginx
location /upstream {
    proxy_pass                https://backend.example.com;
    proxy_ssl_certificate     /etc/nginx/client.pem;
    proxy_ssl_certificate_key /etc/nginx/client.key;
}
```

Looks simple enough, but alas, they didn't work out of the box for me... This is a post recounting my troubleshooting efforts and learning experience on this matter.

**tl;dr**:

> - [Securing HTTP Traffic to Upstream Servers](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/) provides a good starting point for NGINX mTLS origination
> - NGINX does not include [Server Name Indication (SNI)](https://en.wikipedia.org/wiki/Server_Name_Indication) in the **Client Hello** message by default
> - NGINX reuses the TLS connection to an upstream by default to reduce the amount of TLS handshakes

# Knock, knock

To help test mTLS origination on NGINX, I found a public endpoint [https://certauth.idrix.fr/json/](https://certauth.idrix.fr/json/) (thanks [stranger on the Internet](https://stackoverflow.com/a/66824988)!) that:
1. accepts certificates presented by clients and returns `200`, along with the contents of the client certificates in the body
1. returns `403` when no client certificate is presented. The important thing to note here is that the server will still accept a plain TLS connection and responds with an error at the HTTP level

I started with this `nginx.conf`:
```nginx
events {}
http {
    upstream backend {
        server certauth.idrix.fr:443;
    }

    # Server block that presents client certificate to backend. A 200 response is expected.
    server {
        listen 8080;

        location / {
            # Specify client key pair for mTLS
            proxy_ssl_certificate       /etc/nginx/certs/client.crt;
            proxy_ssl_certificate_key   /etc/nginx/certs/client.key;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }

    # Server block that DOES NOT presents client certificate to backend. A 403 response is expected.
    server {
        listen 8081;

        location / {
            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }
}
```

It has two `server` blocks - one uses a client key pair in the connection towards the upstream (`:8080`), and another uses plain TLS (`:8081`). Both endpoints listen for plain HTTP traffic on the client side, essentially offloading traffic **encryption** for the client towards the upstream.

It immediately fell apart 😢. Hitting the `:8080` mTLS endpoint on the NGINX returned a `403`
```sh
$ curl -vvv localhost:8080/json/
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 11:12:10 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
```

# Who's there?

Thinking that there might be issues with my certificate, I tried hitting the upstream directly with `curl`, referencing the same client key pair passed to the NGINX directives. That got through fine.
```sh
$ curl --key $(pwd)/.certs/client.key --cert $(pwd)/.certs/client.crt https://certauth.idrix.fr/json/
{"LANG":"C.UTF-8","INVOCATION_ID":"a7f615d74e904aa5a75d575c53d8c316","HTTP_ACCEPT":"*\/*","HTTP_USER_AGENT":"curl\/7.68.0","HTTP_HOST":"certauth.idrix.fr","SSL_CLIENT_I_DN":"CN=rootCA.test,O=NGINX-mtls,C=AU","SSL_CLIENT_S_DN":"CN=client,C=AU","SSL_CLIENT_VERIFY":"FAILED:unable to verify the first certificate","SSL_CLIENT_V_END":"Mar 27 22:46:00 2023 GMT","SSL_CLIENT_V_START":"Mar 27 22:46:00 2022 GMT","SSL_CLIENT_SERIAL":"54229823E16037FBDC4B764E4B86BC8D34536302","SSL_CLIENT_FINGERPRINT":"8a8e6e240075445cef6bfdfcef65d0504f76f707","SSL_SESSION_ID":"5995567bc6aba7a075b460a1e71dd6593fdcc260673c579540666a88ae7e0f0e","SSL_SERVER_NAME":"certauth.idrix.fr","SSL_CIPHER":"ECDHE-RSA-AES256-GCM-SHA384","SSL_PROTOCOL":"TLSv1.2","HTTPS":"on","PATH_INFO":"","SERVER_NAME":"certauth.idrix.fr","SERVER_PORT":"443","SERVER_ADDR":"54.36.191.227","REMOTE_PORT":"49916","REMOTE_ADDR":"121.208.219.76","SERVER_PROTOCOL":"HTTP\/1.1","DOCUMENT_URI":"\/json\/index.php","REQUEST_URI":"\/json\/","CONTENT_LENGTH":"","CONTENT_TYPE":"","REQUEST_METHOD":"GET","QUERY_STRING":"","REQUEST_TIME_FLOAT":1648552481.956386,"REQUEST_TIME":1648552481}
```

I started doubting the [proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate) and [proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key) directives, and went off to reproduce the results shown in the NGINX [write-up](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/) with a self hosted mTLS NGINX server. And of course the directives worked fine. So far I've established that the following were working correctly:
- client certificates
- [proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate) and [proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key) directives
- the test endpoint [certauth.idrix.fr])(https://certauth.idrix.fr/json/)

That leaves my NGINX configuration that dictated the connection setup towards the upstream. Grasping at straws, I decided to break out [Wireshark](https://www.wireshark.org/) and capture the mTLS handshakes initiated by NGINX and `curl` for comparison:

| NGINX | curl |
| :-: | :-: |
| ![Without SNI](/images/nginx-mtls-origination-no-sni.png) | ![With SNI](/images/nginx-mtls-origination-curl-sni.png) |

And there it was - the [Server Name Indication (SNI)](https://en.wikipedia.org/wiki/Server_Name_Indication) was missing in the **Client Hello** message sent by NGINX to the [certauth.idrix.fr](https://certauth.idrix.fr). Often, web servers route traffic to different services based on SNI for traffic over TLS connections. A quick look on NGINX documentation revealed the [proxy_ssl_server_name](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_server_name) and [proxy_ssl_name](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_name) directives which allow us to specify the SNI in the TLS handshake towards the upstream, bringing the NGINX configuration to:

```nginx
events {}
http {
    upstream backend {
        server certauth.idrix.fr:443;
    }

    # Server block that presents client certificate to backend. A 200 response is expected.
    server {
        listen 8080;

        location / {
            # Specify client key pair for mTLS
            proxy_ssl_certificate       /etc/nginx/certs/client.crt;
            proxy_ssl_certificate_key   /etc/nginx/certs/client.key;

            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }

    # Server block that DOES NOT presents client certificate to backend. A 403 response is expected.
    server {
        listen 8081;

        location / {
            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }
}
```

Sending a HTTP request to the mTLS endpoint `:8080` on the NGINX returned `200` and the client certificate details in the response body. Great!
```sh
$ curl -vvv localhost:8080/json/
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 11:46:34 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
<
{"LANG":"C.UTF-8","INVOCATION_ID":"a7f615d74e904aa5a75d575c53d8c316","HTTP_ACCEPT":"*\/*","HTTP_USER_AGENT":"curl\/7.68.0","HTTP_CONNECTION":"close","HTTP_HOST":"certauth.idrix.fr","SSL_CLIENT_I_DN":"CN=rootCA.test,O=NGINX-mtls,C=AU","SSL_CLIENT_S_DN":"CN=client,C=AU","SSL_CLIENT_VERIFY":"FAILED:unable to verify the first certificate","SSL_CLIENT_V_END":"Mar 27 22:46:00 2023 GMT","SSL_CLIENT_V_START":"Mar 27 22:46:00 2022 GMT","SSL_CLIENT_SERIAL":"54229823E16037FBDC4B764E4B86BC8D34536302","SSL_CLIENT_FINGERPRINT":"8a8e6e240075445cef6bfdfcef65d0504f76f707","SSL_SERVER_NAME":"certauth.idrix.fr","SSL_CIPHER":"ECDHE-RSA-AES256-GCM-SHA384","SSL_PROTOCOL":"TLSv1.2","HTTPS":"on","PATH_INFO":"","SERVER_NAME":"certauth.idrix.fr","SERVER_PORT":"443","SERVER_ADDR":"54.36.191.227","REMOTE_PORT":"49836","REMOTE_ADDR":"121.208.219.76","SERVER_PROTOCOL":"HTTP\/1.0","DOCUMENT_URI":"\/json\/index.php","REQUEST_URI":"\/json\/","CONTENT_LENGTH":"","CONTENT_TYPE":"","REQUEST_METHOD":"GET","QUERY_STRING":"","REQUEST_TIME_FLOAT":1648554395.032386,"REQUEST_TIME":1648554395}
* Connection #0 to host localhost left intact
```

# Oh, it's you again

Time to finish this off by confirming that a plain TLS connection (`:8081`) to [certauth.idrix.fr](https://certauth.idrix.fr) should be rejected...
```sh
$ curl -vvv localhost:8081/json/
*   Trying 127.0.0.1:8081...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 11:52:34 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
<
{"LANG":"C.UTF-8","INVOCATION_ID":"a7f615d74e904aa5a75d575c53d8c316","HTTP_ACCEPT":"*\/*","HTTP_USER_AGENT":"curl\/7.68.0","HTTP_CONNECTION":"close","HTTP_HOST":"certauth.idrix.fr","SSL_CLIENT_I_DN":"CN=rootCA.test,O=NGINX-mtls,C=AU","SSL_CLIENT_S_DN":"CN=client,C=AU","SSL_CLIENT_VERIFY":"FAILED:unable to verify the first certificate","SSL_CLIENT_V_END":"Mar 27 22:46:00 2023 GMT","SSL_CLIENT_V_START":"Mar 27 22:46:00 2022 GMT","SSL_CLIENT_SERIAL":"54229823E16037FBDC4B764E4B86BC8D34536302","SSL_CLIENT_FINGERPRINT":"8a8e6e240075445cef6bfdfcef65d0504f76f707","SSL_SESSION_ID":"435cec1dbd4617ecb0d20720a4cabd65b48796ef11eda19e0da05a00779abee2","SSL_SERVER_NAME":"certauth.idrix.fr","SSL_CIPHER":"ECDHE-RSA-AES256-GCM-SHA384","SSL_PROTOCOL":"TLSv1.2","HTTPS":"on","PATH_INFO":"","SERVER_NAME":"certauth.idrix.fr","SERVER_PORT":"443","SERVER_ADDR":"54.36.191.227","REMOTE_PORT":"49870","REMOTE_ADDR":"121.208.219.76","SERVER_PROTOCOL":"HTTP\/1.0","DOCUMENT_URI":"\/json\/index.php","REQUEST_URI":"\/json\/","CONTENT_LENGTH":"","CONTENT_TYPE":"","REQUEST_METHOD":"GET","QUERY_STRING":"","REQUEST_TIME_FLOAT":1648554754.749326,"REQUEST_TIME":1648554754}
* Connection #0 to host localhost left intact
```

Um... I received a `200` and could see the details of client certificate in the response body, even though NGINX was not supposed to send the client certificate ti the upstream!? I decided to restart NGINX and reattempt the request
```sh
$ curl -vvv localhost:8081/json/
*   Trying 127.0.0.1:8081...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 11:58:09 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
```

This time the request was rejected, but so did a subsequent request to the mTLS endpoint `:8080`!
```sh
$ curl -vvv localhost:8080/json/
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 11:58:43 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
```

I just proved earlier that mTLS origination worked! Something was suspicious about the connection between NGINX and the upstream, it looked like the connection was being reused by the two `server` blocks even though they were configured differently.

Fortunately for me, I was able to quickly find an aptly named directive [proxy_ssl_session_reuse](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse), which by default tells NGINX to reuse SSL/TLS sessions towards the upstream.

> You'd probably want to leave this enabled in most cases to reduce the amount of TLS handshakes, unless you have differing TLS settings for each request (another example I can think of is presenting dynamic client certificates to the upstream).

I updated the NGINX configuration to disable this behaviour:

```nginx
events {}
http {
    upstream backend {
        server certauth.idrix.fr:443;
    }

    # Server block that presents client certificate to backend. A 200 response is expected.
    server {
        listen 8080;

        location / {
            # Specify client key pair for mTLS
            proxy_ssl_certificate       /etc/nginx/certs/client.crt;
            proxy_ssl_certificate_key   /etc/nginx/certs/client.key;

            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }

    # Server block that DOES NOT presents client certificate to backend. A 403 response is expected.
    server {
        listen 8081;

        location / {
            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }
}
```

and now for the test:
```sh
$ curl -vvv localhost:8080/json/
*   Trying 127.0.0.1:8080...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 12:04:59 GMT
< Content-Type: application/json
< Transfer-Encoding: chunked
< Connection: keep-alive
<
{"LANG":"C.UTF-8","INVOCATION_ID":"a7f615d74e904aa5a75d575c53d8c316","HTTP_ACCEPT":"*\/*","HTTP_USER_AGENT":"curl\/7.68.0","HTTP_CONNECTION":"close","HTTP_HOST":"certauth.idrix.fr","SSL_CLIENT_I_DN":"CN=rootCA.test,O=NGINX-mtls,C=AU","SSL_CLIENT_S_DN":"CN=client,C=AU","SSL_CLIENT_VERIFY":"FAILED:unable to verify the first certificate","SSL_CLIENT_V_END":"Mar 27 22:46:00 2023 GMT","SSL_CLIENT_V_START":"Mar 27 22:46:00 2022 GMT","SSL_CLIENT_SERIAL":"54229823E16037FBDC4B764E4B86BC8D34536302","SSL_CLIENT_FINGERPRINT":"8a8e6e240075445cef6bfdfcef65d0504f76f707","SSL_SERVER_NAME":"certauth.idrix.fr","SSL_CIPHER":"ECDHE-RSA-AES256-GCM-SHA384","SSL_PROTOCOL":"TLSv1.2","HTTPS":"on","PATH_INFO":"","SERVER_NAME":"certauth.idrix.fr","SERVER_PORT":"443","SERVER_ADDR":"54.36.191.227","REMOTE_PORT":"49922","REMOTE_ADDR":"121.208.219.76","SERVER_PROTOCOL":"HTTP\/1.0","DOCUMENT_URI":"\/json\/index.php","REQUEST_URI":"\/json\/","CONTENT_LENGTH":"","CONTENT_TYPE":"","REQUEST_METHOD":"GET","QUERY_STRING":"","REQUEST_TIME_FLOAT":1648555499.597671,"REQUEST_TIME":1648555499}
* Connection #0 to host localhost left intact

$ curl -vvv localhost:8081/json/
*   Trying 127.0.0.1:8081...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 8081 (#0)
> GET /json/ HTTP/1.1
> Host: localhost:8081
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 403 Forbidden
< Server: nginx/1.21.3
< Date: Tue, 29 Mar 2022 12:05:03 GMT
< Content-Type: text/html; charset=UTF-8
< Transfer-Encoding: chunked
< Connection: keep-alive
<
* Connection #0 to host localhost left intact
```

Huzzah! Both mTLS and simple TLS endpoints on the NGINX worked as intended.

It was fun detective work figuring this out. Now to port some of this into my NGINX Service Mesh deployment. Stay tuned!
