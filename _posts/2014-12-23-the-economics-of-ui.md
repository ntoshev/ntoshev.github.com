---
layout: post
title: "The Economics of UI"
date: 2014-12-23 14:00:08 +0200
categories: 
---
*(and What To Do About It)*

## Cost at scale
Software is famous for having marginal cost of essentially zero (the marginal cost is the cost of producing one more unit after the up-front cost, like building a factory, has been payed). This has big implications:

- a natural winner-take-all dynamics of markets
- venture capital ecosystem works
- open source is economically possible

> *Note:* If you believe that [software is eating the world](http://www.wsj.com/articles/SB10001424053111903480904576512250915629460), this has interesting implications about the way the world of atoms is going to change soon. With software becoming a bigger component of physical goods and services, the fixed cost component to them is going to increase and the marginal cost is going to decrease. This means we might start to see some of the software-world dynamics (winner-take-all, venture capital, maybe even open source) in the physical economics as well. Uber and Tesla may be just precursors to a larger trend.

## UI

Unlike the rest of the software, user interface has substantial marginal cost. An interface has two sides, UI has a software and a human side, each having different cost characteristics:

1. The software side has large fixed cost to build, and zero marginal cost.
2. The human side is the **cost to adapt your users' brains** to the way the software side of your UI works. The up-front cost here is e.g. the cost of creating the user manual, the marginal cost is training, support, or hidden in the smaller viral coefficient of your consumer app.

This understanding allows you to make decisions if and when to invest more in the software side of good UI, in order to reduce the marginal cost on the human side. It also means having a minimal UI is a good thing all else being equal.

## Modeling a domain

Typical software models a part of the world with a set of hard rules. The user manipulates this formal model directly applying these rules via the user interface. Secondary effects from user actions are inferred by software and reflected on the model.

For example, in word processing, you get to manipulate text at higher level (type and edit text, apply formatting), with software inferring low level details (line breaks at page width, text flow around images, repeating header/footer etc). The computer maintains a model of text and applies deterministic operations on it. The model is fully transparent to the user, in fact he needs to understand it in order to use the software. After almost 40 years evolution and integration in the mainstream writing culture, the basic operations (typing and editing, line breaks, bold/italic/font/size formatting) feel natural. More complex things like maintaining consistent styling in a document are harder.

## How to make minimal UI

One way to make UI (or software) more useful is to have a richer formal model that infers and silently does many useful things: like a spreadsheet does formula recalculation, or like databases utilize indices and nontrivial plan optimizations to execute queries. Unfortunately, good formal models are few and far between.

When a nice formal theory of the domain is not available, you can still fake it by inferring side effects with a predictive model, or machine learning. For example, a word processing program might try to infer what is the consistent styling that the user wants and apply it. A good UI might be one that hides part of the internally used model and ask the user to provide only examples of what he wants done. If the inference is good enough, that simplifies the user's mental model, so it reduces the marginal cost of your software.