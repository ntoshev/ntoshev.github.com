---
layout: post
title: "Monkey Patching Considered Harmless (When Done Right)"
date: 2014-07-24
categories: 
published: true
---

Monkey patching is universally considered a bad practice. Many people avoid having anything to do with it (and with projects that use it, like [gevent](http://www.gevent.org/)), without considering if the actual reasons for avoiding it apply to their case. So what are these reasons? I think the important ones are:

1. The patched code works very differently from the unpatched, breaking assumptions the rest of the code has about it.

2. Monkey patches making assumptions about the code they patch that are implementation detail and change between versions

(Aside: Most nontrivial errors are about programmer coding wrong assumptions. These can be low level assumptions about how the code behaves or higher level assumptions about the domain that is being modeled.)

Monkey patching is too general so it's easy to slip into one of these. Note that the same applies about things like inheritance (vs composition). You can avoid these problems with monkey patching if you:

> Write code to monkey patch functions or methods as decorators. Make that decorator general, i.e. assume as little about the modified code as possible. 

It is the second sentence that takes care of the problems above, but it is the decorators that make it possible to write such code (this is not strictly true, any higher order function will do, but decorators provide a nice convention with syntax sugar as an added bonus). We can apply Python decorators on existing functions and methods to permanently modify their behavior. Let's see how it's done concretely.

## Decorators

If you haven't actually created decorators, you may think about them as magic syntax sugar and not realize what decorators are actually designed to do is to be functions that take another function as an argument and modify it's behavior. However, decorators are usually applied at the moment of defining the function they apply to, this is when their special syntax is designed to be used:

```python
@decorator
def func(arg1, arg2):
    pass
```

Without the syntax sugar, the way to do the same would be (note you can do this also in e.g. Javascript, so this style of monkey patching is available to you in other dynamic languages besides Python):

```python
def func(arg1, arg2):
    pass
func = decorator(func)
```

If the last line is separate from the function declaration, and the function wasn't specifically written to be decorated, you will effectively do monkey patching. Some decorators (e.g. doing logging) don't make any assumptions about the functions they decorate, and the decorated functions could be used without any decorator. Others (like [bottlepy's routing decorators](http://bottlepy.org/docs/0.11/tutorial.html#request-routing) or `memoize` that you'll see bellow) make only a few assumptions. 

If you do monkey patching with decorators, you avoid the problems with them, because decorators are written to modify the function's behavior in an orthogonal way. Let's see a specific real-world example.

## High volume HTTP requests and DNS resolution

A common gotcha when writing a crawler is that the bottleneck you will likely hit first is the DNS resolution. I also hit this problem when prototyping https://t1mr.com (pronounced 'timer', a website availability tool whose basic function is to make an http request to your site every minute and notify you if it's down). It is weird to diagnose because, depending on your DNS infrastructure, you can get various failures:

 * your home / office router may crash
 * the DNS infrastructure of your hosting provider may start to return random errors for some of the requests - some timeouts, some other failures (I experienced this with both Hetzner and Amazon EC2)
 * [Google's Public DNS service](https://developers.google.com/speed/public-dns/) would actually behave quite nicely by servicing all your requests, although rate limiting them to 10 resolutions per second.

I eventually settled to use Google's DNS because it's solid and predictable. But I wanted to make the basic service for t1mr free, so it had to be cheap to run and scale well on a single machine. 10 req/second are just 600 per minute, and I really wanted to saturate the network link.

Normally one would set up a DNS cache on his server and use that. I didn't quite get this to work (perhaps I exhausted the cache), but I didn't really try because I wanted maximum control over when and how the domain names are resolved. So how can you implement such a cache in the application?

DNS resolutions are normally handled by the OS. When you open a socket to a remote server specified by name, the  OS blocks that system call and resolves it (unless it is already cached). Resolution can take 100s of milliseconds, that's why high level frameworks that allow non-blocking code (Python with gevent or node.js) has their own way of doing it (they both use c-ares or blocking OS syscalls in threadpools depending on configuration).

You can modify the http call to use an IP address and set the Host header appropriately. This works, although you have to do it yourself because http libraries typically lack support for it. But in t1mr.com I want to provide checking other types of services besides http, say ICMP ping or SMTP check. The other protocols lack the option to use an IP and specify the domain name separately.

What I did instead was monkey-patch [`getaddrinfo`(the function that resolves domain names)](https://docs.python.org/2/library/socket.html#socket.getaddrinfo) with a memoization decorator. Here is a generic memoizer: 

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

In production we use gevent for efficient networking and a different decorator (that behaves mostly like a cache with expiration). Any networking code will now use under the hood our modified `getaddrinfo`, resolve the domain name only the first time and use a cached value from now on (unless the memoizer decides it is time to refresh).

This might seem like an isolated example where it's a good idea to patch lower level function, but it's not.

## Layered abstractions

Basically all computing is built on layers upon layers of abstractions. It is considered a good idea to implement a layer relying only upon the layer bellow, and it it. Networks are probably the canonical example. One of the actual limits of this model is that you have to expose the low level abstractions to the high level for tweaking. If you don't have a mechanism to do this, low level abstractions often leak in a way that is not fixable from the high level code. Monkey patching the lower levels is not a panacea, but it can help here.

## Conclusion

Monkey patching is seen as an ugly kludge, but is still used quite a lot because people lack better ways to accomplish their goals using the abstractions available in their language. One way to do safe and useful monkey patching is to write the monkey patch as a decorator, making very few assumptions about the function being decorated, and get a nice orthogonal abstraction in this way.