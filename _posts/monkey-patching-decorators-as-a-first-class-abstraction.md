---
layout: post
title: "Monkey patching decorators as a first class abstraction"
date: 2014-05-04
categories: 
published: false
---

Monkey patching is universally considered a bad practice. Nevertheless, people use it because it fills a gap in the common abstractions available in programming. I'll show you such cases and present certain types of monkey patching that are mostly safe.

## High volume HTTP requests and DNS resolution

A common gotcha when writing a crawler is that the bottleneck you will likely hit first is the DNS resolution. I also hit this problem when prototyping https://t1mr.com (pronounced 'timer', a website availability tool whose basic function is to make an http request to your site every minute and notify you if it's down). It is weird to diagnose because, depending on your DNS infrastructure, you can get various failures:
 * your home / office router may crash
 * the DNS infrastructure of your hosting provider may start to return random errors for some of the requests - some timeouts, some other failures (I experienced this with both Hetzner and Amazon EC2)
 * [Google's Public DNS service](https://developers.google.com/speed/public-dns/) would actually behave quite nicely by servicing all your requests, although rate limiting them to 10 resolutions per second.

I've seen all of the above, and eventually settled to use Google's DNS because it's solid and predictable. But I wanted to make the basic service for t1mr free, so it had to be cheap to run and scale well on a single machine. 10 req/second are just 600 per minute, and I really wanted to saturate the network link.

Normally one would set up a DNS cache on his server and use that. I didn't quite get this to work (perhaps I exhausted the cache), but I didn't really try because I wanted maximum control over when and how the domain names are resolved. So how can you do such a thing in the application?

DNS resolutions are normally handled by the OS. When you open a socket to a remote server specified by name, the  OS blocks that system call and resolves it (unless it is already cached). Resolution can take 100s of milliseconds, that's why high level frameworks that allow non-blocking code (Python with gevent or node.js) has their own way of doing it (they both use c-ares or blocking OS syscalls in threadpools depending on configuration).

You can modify the http call to use an IP address and set the Host header appropriately. This works, although you have to do it yourself because http libraries typically lack support for it. But in t1mr.com I want to provide checking other types of services besides http, say ICMP ping or SMTP check. The other protocols lack the option to use an IP and specify the domain name separately.

What I did instead was monkey-patch getaddrinfo with memoization decorator. Here is a generic memoizer: 

```python
def memoize(f):
  global cache
  cache= {}
  def memf(*x):
      if x not in cache:
          cache[x] = f(*x)
      return cache[x]
  return memf
```
It is very general and can be used with pretty much any function, like: 

```python
@memoize
def fib(n):
	if n<2:
		return 1
	else:
		return fib(n-1)+fib(n-2)
```
In our case, it is good enough for prototyping. Here is how to monkey-patch `socket.getaddrinfo` with it:

```python
import socket
socket.getaddrinfo = memoize(socket.getaddrinfo)
```

In production we use gevent for efficient networking and a different decorator. Any networking code will now use under the hood our modified `getaddrinfo`, resolve the domain name only the first time and use a cached value from now on.

This might seem like an isolated example where it's a good idea to patch lower level function, but it's not.

## Layered abstractions

Basically all computing is built on layers upon layers of abstractions. It is considered a good idea to implement a layer relying only upon the layer bellow, and it it. Net