---
layout: post
title: "What underengineering?"
date: 2014-04-29 17:03:08 +0200
categories: 
---

A lot of the troubles in software development I've seen stem from putting too much structure in your code. Too many layers of indirection, too much meta, too many classes, too much inheritance, too many FactoryFactories. In a series of posts I'll attempt to show a different way: building small, orthogonal (arbitrary composable) abstractions and using them to do your job with minimal amount of code. The code you write with them should express your intent directly, in the problem domain, not hide it behind some incidental structure of the programming domain. 

Here are some examples of beautiful code in Python that are not overengineered:

* [Peter Norvig](http://norvig.com/)'s [public](http://norvig.com/ngrams/) [code](http://norvig.com/sudoku.html) and [CS212](https://www.udacity.com/course/cs212‎)
* [Tornado Web Framework](https://github.com/facebook/tornado)
* [Bottle](https://github.com/defnull/bottle)

