---
title: "Explicit and Transparent Forward Proxies"
date: 2023-03-05T20:12:03+11:00
draft: false
tags: ['proxy', 'forward-proxy', 'squid']
categories: ['tech']
---

---

Forward proxies, by definition, initiates requests to servers on behalf of the clients. Common reasons to use forward proxies include:
1. hiding client's IP address from the server to provide a layer of security,
1. restricting access to certain websites,
1. performing content filtering, and
1. caching frequently accessed content (though this is less irrelevant in the age of dynamically generated sites and high speed Internet access)

<p style="text-align:center;"><img src="/images/explicit-and-transparent-forward-proxies/banner.png" width="56%"></p>

Forward proxies can be set up in two different modes - [explicit](#explicit-mode) and [transparent](#transparent-mode) mode. In this post, I will be explaining both modes and how they handle HTTP and HTTPS traffic.

---

# Explicit mode

## HTTP server

Let's begin by looking at a client accessing the server directly.

![No proxy](/images/explicit-and-transparent-forward-proxies/no_proxy.png)

The server is able to see the client's IP in the source address, and the HTTP message is a simple `GET /` string.

Explicit proxies require a client to be configured to forward its requests to the proxy. One way of achieving this is by setting the proxy IP and port in the `HTTP_PROXY` or `HTTPS_PROXY` environment variables. The following example shows a client with the `HTTP_PROXY` environment variable set to `http://proxy.example:3128`, and wants to browse to a HTTP endpoint:

![Explicit HTTP](/images/explicit-and-transparent-forward-proxies/explicit-http.png)

When the client sends a HTTP request, it sets up the TCP connection to the proxy and sends a `GET http://<domain>/` request to it.

> Notice the addition of the server domain in the message, an indication that the client thinks its speaking to the server through a proxy.

The proxy performs a DNS lookup on the server domain name, and then uses its own IP address to initiate a separate TCP and HTTP connection to the server. As both requests and responses are unencrypted (a rare sight these days), this scenario enables the proxy to perform filtering based on URL and content, or even modify the content.

## HTTPS server

For connections to a HTTPS server, the client should have the `HTTPS_PROXY` environment variable set to `http://proxy.example:3128`.

The client sets up a TCP connection to the proxy and sends a `CONNECT <domain>:443`. The proxy looks up the domain and establishes a TCP session to the server. With the TCP session up, the proxy sends a `HTTP/1.1 200 Connection established` message back to the client, indicating that it is ready to tunnel subsequent traffic from the client to the server.

![Explicit HTTPS TCP proxy](/images/explicit-and-transparent-forward-proxies/explicit-https-tcp-proxy.png)

Next, the client and server establishes a TLS session between themselves through the tunnel set up earlier.

![Explicit HTTPS TLS](/images/explicit-and-transparent-forward-proxies/explicit-https-tls.png)

With the TLS session up, traffic flows between the client and the server fully encrypted, limiting the filtering capability of the proxy to be based just on domain names extracted from Server Name Indication (SNI) field in the `Client Hello` packet or the server certificate presented in `Server Hello`.

### Man In The Middle (MITM)

In some cases, the proxy could be configured to intercept the TLS session. The proxy responds to the `Client Hello` message with its own `Server Hello`, and on the other end, initiates another TLS session with its own `Client Hello` towards the server. This is known as **Man in the middle (MITM)**.

![Explicit HTTPS MITM](/images/explicit-and-transparent-forward-proxies/explicit-https-mitm.png)

As the proxy likely does not hold the original server's TLS private key, it has to generate a fake server certificate with a custom Certificate Authority (CA) to complete the handshake towards the client. If the custom CA is not trusted by the client, the client will detect that the server certificate is fake and be aware of potential MITM attacks.

## Proxy listening on HTTPS port

The above examples have the proxy listening on a HTTP endpoint `http://proxy.example:3128`. This means that initial setup flows between the client and the proxy, such as the `HTTP CONNECT` message, is in clear text, potentially leaking sensitive information to others on the same network. To prevent that, the traffic between client and proxy can be encrypted via TLS with appropriate configuration on both the client and the proxy.

![Explicit Encrypted](/images/explicit-and-transparent-forward-proxies/explicit-encrypted.png)

Once the TLS session has been setup, the message flow for both HTTP and HTTPS traffic from client to proxy to server works as described in the previous sections.

---

# Transparent mode

With explicit mode, a client is aware that it is speaking to a server through a proxy. It requires explicit configuration of the proxy on the client, which is not always supported.

Transparent mode is another deployment option where client traffic is redirected to the proxy by the underlying networking stack, without the client applications being aware. The redirection could be achieved via methods such as

1. configuring the proxy IP to be the gateway on the client, or
1. configuring port forwarding on a router/gateway to redirect all traffic from a client destined for port 80/443 traffic, to the proxy IP instead.

![Transparent](/images/explicit-and-transparent-forward-proxies/transparent.png)

In any case, the traffic lands on the proxy without explicit interactions with the client such as the `HTTP CONNECT` messages. The proxy then processes the HTTP messages similar to  that in explicit mode:

- For HTTP traffic, the proxy passes requests and responses between the client and server, with full visibility into the payload.
- For HTTPS traffic, the proxy either acts as a TCP proxy where its visibility into traffic stops after the initial TLS handshake between the client and the server, or be a MITM to perform more advanced functions like URL and content filtering.

---

# Closing

Which mode to choose will depend on whether your application supports explicit configuration of a HTTP/HTTPS proxy. And if you require the proxy to perform deeper inspection of HTTPS payload, don't forget to consider the trust issues with the proxy generating fake server certificates to intercept the TLS sessions.

If you want to get started with forward proxies, consider looking at [Squid](http://www.squid-cache.org/) or [Tinyproxy](https://tinyproxy.github.io/). For reference, here is a link to [an example of Squid proxy being configured as a transparent proxy](https://github.com/leonseng/squid-transparent-proxy-example).
