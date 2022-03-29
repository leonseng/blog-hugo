---
title: "NGINX mTLS Origination"
date: 2022-03-29T21:46:13+11:00
draft: false
tags: ['nginx', 'mtls']
categories: ['tech']
---

---

Whilst testing some egress control capabilities of [NGINX Service Mesh](https://github.com/nginxinc/nginx-service-mesh) x [NGINX Ingress Controller](https://github.com/nginxinc/kubernetes-ingress) (more on them in the future 😉), I had to configure NGINX to perform mutual TLS origination towards an upstream test server. The official NGINX documentation has a write-up on [Securing HTTP Traffic to Upstream Servers](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/), detailing how this can be achieved with the [proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate) and [proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key) directives:

```nginx
location /upstream {
    proxy_pass                https://backend.example.com;
    proxy_ssl_certificate     /etc/nginx/client.pem;
    proxy_ssl_certificate_key /etc/nginx/client.key;
}
```

Looks simple enough, but alas, they didn't work out of the box for me... This is a post recounting my troubleshooting efforts and some NGINX knobs I learned along the way.

**tl;dr**:

> - [Securing HTTP Traffic to Upstream Servers](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/) provides a good starting point for NGINX mTLS origination
> - NGINX does not include [Server Name Indication (SNI)](https://en.wikipedia.org/wiki/Server_Name_Indication) in the **Client Hello** message by default
> - NGINX reuses the TLS connection to an upstream by default to reduce the amount of TLS handshakes

# Knock, knock - the setup

To help test mTLS origination on NGINX, I found a public endpoint [https://certauth.idrix.fr/json/](https://certauth.idrix.fr/json/) (thanks [stranger on the Internet](https://stackoverflow.com/a/66824988)!) that:
1. accepts certificates presented by clients and returns `200`, along with the contents of the client certificates in the body, notably the `SSL_CLIENT_S_DN` field
    ```sh
    $ curl -vvvs --key client.key --cert client.crt https://certauth.idrix.fr/json/ | jq .
    *   Trying 54.36.191.227:443...
    * TCP_NODELAY set
    * Connected to certauth.idrix.fr (54.36.191.227) port 443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *   CAfile: /etc/ssl/certs/ca-certificates.crt
    CApath: /etc/ssl/certs
    } [5 bytes data]
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    } [512 bytes data]
    * TLSv1.3 (IN), TLS handshake, Server hello (2):
    { [112 bytes data]
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    { [4026 bytes data]
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    { [300 bytes data]
    * TLSv1.2 (IN), TLS handshake, Request CERT (13):
    { [52 bytes data]
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    { [4 bytes data]
    * TLSv1.2 (OUT), TLS handshake, Certificate (11):
    } [747 bytes data]
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    } [37 bytes data]
    * TLSv1.2 (OUT), TLS handshake, CERT verify (15):
    } [264 bytes data]
    * TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
    } [1 bytes data]
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    } [16 bytes data]
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    { [16 bytes data]
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
    * ALPN, server accepted to use http/1.1
    * Server certificate:
    *  subject: CN=certauth.idrix.fr
    *  start date: Jan 25 00:49:08 2022 GMT
    *  expire date: Apr 25 00:49:07 2022 GMT
    *  subjectAltName: host "certauth.idrix.fr" matched cert's "certauth.idrix.fr"
    *  issuer: C=US; O=Let's Encrypt; CN=R3
    *  SSL certificate verify ok.
    } [5 bytes data]
    > GET /json/ HTTP/1.1
    > Host: certauth.idrix.fr
    > User-Agent: curl/7.68.0
    > Accept: */*
    >
    { [5 bytes data]
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 200 OK
    < Server: nginx/1.18.0 (Ubuntu)
    < Date: Tue, 29 Mar 2022 22:47:56 GMT
    < Content-Type: application/json
    < Transfer-Encoding: chunked
    < Connection: keep-alive
    <
    { [1138 bytes data]
    * Connection #0 to host certauth.idrix.fr left intact
    {
    "LANG": "C.UTF-8",
    "INVOCATION_ID": "a7f615d74e904aa5a75d575c53d8c316",
    "HTTP_ACCEPT": "*/*",
    "HTTP_USER_AGENT": "curl/7.68.0",
    "HTTP_HOST": "certauth.idrix.fr",
    "SSL_CLIENT_I_DN": "CN=rootCA.test,O=NGINX-mtls,C=AU",
    "SSL_CLIENT_S_DN": "CN=client,C=AU",
    "SSL_CLIENT_VERIFY": "FAILED:unable to verify the first certificate",
    "SSL_CLIENT_V_END": "Mar 27 22:46:00 2023 GMT",
    "SSL_CLIENT_V_START": "Mar 27 22:46:00 2022 GMT",
    "SSL_CLIENT_SERIAL": "54229823E16037FBDC4B764E4B86BC8D34536302",
    "SSL_CLIENT_FINGERPRINT": "8a8e6e240075445cef6bfdfcef65d0504f76f707",
    "SSL_SESSION_ID": "af166b73a73e3548a53680986a328e55f55f8e48492485d22a8a9e89010587ce",
    "SSL_SERVER_NAME": "certauth.idrix.fr",
    "SSL_CIPHER": "ECDHE-RSA-AES256-GCM-SHA384",
    "SSL_PROTOCOL": "TLSv1.2",
    "HTTPS": "on",
    "PATH_INFO": "",
    "SERVER_NAME": "certauth.idrix.fr",
    "SERVER_PORT": "443",
    "SERVER_ADDR": "54.36.191.227",
    "REMOTE_PORT": "49908",
    "REMOTE_ADDR": "121.208.219.76",
    "SERVER_PROTOCOL": "HTTP/1.1",
    "DOCUMENT_URI": "/json/index.php",
    "REQUEST_URI": "/json/",
    "CONTENT_LENGTH": "",
    "CONTENT_TYPE": "",
    "REQUEST_METHOD": "GET",
    "QUERY_STRING": "",
    "REQUEST_TIME_FLOAT": 1648594076.540687,
    "REQUEST_TIME": 1648594076
    }
    ```
1. returns `403` when no client certificate is presented. The important thing to note here is that the server will still accept a plain TLS connection and responds with an error at the HTTP level
    ```sh
    $ curl -vvvs https://certauth.idrix.fr/json/
    *   Trying 54.36.191.227:443...
    * TCP_NODELAY set
    * Connected to certauth.idrix.fr (54.36.191.227) port 443 (#0)
    * ALPN, offering h2
    * ALPN, offering http/1.1
    * successfully set certificate verify locations:
    *   CAfile: /etc/ssl/certs/ca-certificates.crt
    CApath: /etc/ssl/certs
    * TLSv1.3 (OUT), TLS handshake, Client hello (1):
    * TLSv1.3 (IN), TLS handshake, Server hello (2):
    * TLSv1.2 (IN), TLS handshake, Certificate (11):
    * TLSv1.2 (IN), TLS handshake, Server key exchange (12):
    * TLSv1.2 (IN), TLS handshake, Request CERT (13):
    * TLSv1.2 (IN), TLS handshake, Server finished (14):
    * TLSv1.2 (OUT), TLS handshake, Certificate (11):
    * TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
    * TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
    * TLSv1.2 (OUT), TLS handshake, Finished (20):
    * TLSv1.2 (IN), TLS handshake, Finished (20):
    * SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
    * ALPN, server accepted to use http/1.1
    * Server certificate:
    *  subject: CN=certauth.idrix.fr
    *  start date: Jan 25 00:49:08 2022 GMT
    *  expire date: Apr 25 00:49:07 2022 GMT
    *  subjectAltName: host "certauth.idrix.fr" matched cert's "certauth.idrix.fr"
    *  issuer: C=US; O=Let's Encrypt; CN=R3
    *  SSL certificate verify ok.
    > GET /json/ HTTP/1.1
    > Host: certauth.idrix.fr
    > User-Agent: curl/7.68.0
    > Accept: */*
    >
    * Mark bundle as not supporting multiuse
    < HTTP/1.1 403 Forbidden
    < Server: nginx/1.18.0 (Ubuntu)
    < Date: Tue, 29 Mar 2022 22:49:14 GMT
    < Content-Type: text/html; charset=UTF-8
    < Transfer-Encoding: chunked
    < Connection: keep-alive
    <
    * Connection #0 to host certauth.idrix.fr left intact
    ```

I started with this `nginx.conf` which has two `server` blocks:
- `:8080` establishes mTLS with the client key pair towards the upstream,
- `:8081` uses plain TLS.

Both endpoints listen for plain HTTP traffic on the client side, essentially offloading traffic **encryption** for the client towards the upstream:
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

# Who's there? - SNI routing woe

I knew there are no issues with the client certificates and the test endpoint [certauth.idrix.fr](https://certauth.idrix.fr) in an earlier test. I also managed to replicate the example in the NGINX [write-up](https://docs.nginx.com/nginx/admin-guide/security-controls/securing-http-traffic-upstream/), so that rules out issues with the [proxy_ssl_certificate](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate) and [proxy_ssl_certificate_key](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_certificate_key) directives. This leaves my NGINX configuration that dictated the connection setup towards the upstream. Grasping at straws, I decided to break out [Wireshark](https://www.wireshark.org/) and capture the mTLS handshakes initiated by NGINX and `curl` for comparison:

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

# Oh, it's you again - connection reuse

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

Um... I received a `200` and could see the details of client certificate in the response body, even though NGINX was not supposed to send the client certificate to the upstream!? I decided to restart NGINX and reattempt the request
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

This time the request was rejected, but so did a subsequent request to the mTLS endpoint `:8080`! I swear `:8080` worked earlier, just scroll up!
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

Something was suspicious about the connection between NGINX and the upstream, it looked like the connection was being reused by the two `server` blocks even though they were configured differently.

Fortunately for me, I was able to quickly find an aptly named directive [proxy_ssl_session_reuse](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse), which by default tells NGINX to reuse SSL/TLS sessions towards the upstream.

> You'd probably want to leave this enabled in most cases to reduce the amount of TLS handshakes, unless you have differing TLS settings for each request (another example I can think of is presenting dynamic client certificates to the upstream, which I will cover in the [Appendix](#appendix) below).

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

            # Stop connection reuse
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

            # Stop connection reuse
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

# Appendix

To confirm my understanding of the [proxy_ssl_session_reuse](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse) directive, here's another test where I have two `server` blocks each presenting a client certificate to the upstream.

Starting with [proxy_ssl_session_reuse](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse) enabled:
```
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

            # Stop connection reuse
            # proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }

    # Server block that DOES NOT presents client certificate to backend. A 403 response is expected.
    server {
        listen 8081;

        location / {
            proxy_ssl_certificate       /etc/nginx/certs/another_client.crt;
            proxy_ssl_certificate_key   /etc/nginx/certs/another_client.key;


            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            # Stop connection reuse
            # proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }
}
```

Hitting both `:8080` and `:8081` consecutively returns the same details on the client certificates, indicating that the upstream sees the same connection from NGINX.
```
$ curl -s localhost:8081/json/ | jq . | grep SSL_CLIENT_S_DN
  "SSL_CLIENT_S_DN": "CN=another_client,C=AU",
$ curl -s localhost:8080/json/ | jq . | grep SSL_CLIENT_S_DN
  "SSL_CLIENT_S_DN": "CN=another_client,C=AU",
```

With [proxy_ssl_session_reuse](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ssl_session_reuse) disabled:
```
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

            # Stop connection reuse
            proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }

    # Server block that DOES NOT presents client certificate to backend. A 403 response is expected.
    server {
        listen 8081;

        location / {
            proxy_ssl_certificate       /etc/nginx/certs/another_client.crt;
            proxy_ssl_certificate_key   /etc/nginx/certs/another_client.key;

            # Set SNI in Client Hello
            proxy_ssl_server_name on;
            proxy_ssl_name  certauth.idrix.fr;

            # Stop connection reuse
            proxy_ssl_session_reuse off;

            proxy_set_header Host certauth.idrix.fr;
            proxy_pass https://backend/;
        }
    }
}
```

The responses now show details of the two client certificates as expected:
```
$ curl -s localhost:8081/json/ | jq . | grep SSL_CLIENT_S_DN
  "SSL_CLIENT_S_DN": "CN=another_client,C=AU",
$ curl -s localhost:8080/json/ | jq . | grep SSL_CLIENT_S_DN
  "SSL_CLIENT_S_DN": "CN=client,C=AU",
```
