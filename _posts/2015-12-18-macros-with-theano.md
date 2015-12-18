---
layout: post
title: "Macros with Theano"
date: 2015-12-18 12:29:00 +0200
categories: 
---

I've been playing with Theano lately, implementing various neural networks and other computations to see if they are suitable to Theano's programming model. 

[Wikipedia says:](https://en.wikipedia.org/wiki/Theano_%28software%29) "Theano is a numerical computation library for Python," but I find this misleading - Theano is really a mathematical expression compiler that doesn't have it's own syntax but is hosted in Python instead. This has the interesting consequence that the type of abstractions that works best with Theano is not functions or objects/classes, but macros - you write functions or methods in Python that generate and compose Theano expressions. I'll explain this in more detail in this post. Interestingly, [Google's TensorFlow](https://www.tensorflow.org/) implements the same model, so the thesis of this post should apply to TensorFlow as well.

With Theano the programmer builds a ["computational graph"](http://deeplearning.net/software/theano/tutorial/symbolic_graphs.html) out of symbolic variables like this:

```python
import theano.tensor as T
x = T.iscalar()
y = T.iscalar()
s = x+y
print s.eval({x:2, y:2}) #4
print s.eval({x:1, y:3}) #4
print (2*s).eval({x:2, y:3}) #10
```

Note that the value of `s` is an expression that can be evaluated many times with different input values, and as part of different computations. It essentially holds a directed graph with nodes being the variables and operations/intermediate results, and vertices connecting the arguments to the operations. This graph is then optimized and executed, typically on a GPU - when the data are not scalar values but large vectors, matrices or more generally tensors, the computation can be parallelized yielding 10-20x improvement in compute time versus a CPU.

For a simple computation you write the math expressions directly in Python as above. The practical problem with that is that very soon you find yourself writing similar expressions with minor differences over and over. The code quickly becomes unwieldy - it is as if you are trying to program without functions or objects! There is lots of Theano code like this on GitHub.

An example of repeated code is the code you use to optimize a cost function with gradient descent. You have to specify how to update parameters on every iteration - here is an example with 2 parameters:

```python
lr = 0.1 # learning rate; how much to move the parameters in the direction of the gradient

# theano will warn you that the order of updates is undefined since we use a dict, but the order doesn't matter
updates = { 
	W: W-lr*T.grad(cost, W),
	b: b-lr*T.grad(cost, b)
}
train_model = theano.function([x, y], outputs=cost, updates=updates) 
```

Note that the learning rate `lr` is a Python variable, not a Theano symbolic variable, so it's a constant at the Theano graph compile time (the last line is when compilation happens).

This seems simple enough, but you may have many parameters, and add / remove them many times experimenting, so how could you remove the repetition? How about writing a function that produces all the necessary updates for a list of your model's parameters, used like this:

```python
train_model = theano.function([x, y], outputs=cost, updates=sgd(0.1, cost, W, b))
```

Here is an implementation with a dict comprehension:

```python
def sgd(lr, cost, *vs):
    return {v: v-lr*T.grad(cost, v) for v in vs}
```

Another example is regularization - L2 regularization takes the weight matrices that parametrize your model, and includes the sum of squared values of the weights in the cost function, like this:

```python
cost = loss + 0.01*((W**2).sum() +(W_prime**2).sum())
```

We could generate that term:

```python

def L2(l, *vs):
    return l*T.add(*((v**2).sum() for v in vs))

cost = loss + L2(0.01, W, W_prime)
```

If you went through [Theano's deep learning tutorials](http://deeplearning.net/tutorial/contents.html), you have probably realized that the classes implementing logistic regression, hidden layers, etc. have methods doing exactly that - they generate pieces of the computational graph that are composed together to produce the complete model.

Since Theano's computational graph contains all the information about its inputs (it's a data structure similar to a compiler's internal representation of a program, but a structure we can traverse), we could find out our parameters dynamically from the cost expression itself, without specifying it explicitly. Here is a function that finds out the variables of specific type (usually `theano.tensor.SharedVariable`, which is the type of vars Theano keeps state in) that your expression depends on:

```python
def find_vars(exp, var_type=T.sharedvar.SharedVariable):
	assert 'owner' in exp, "exp must be a Theano expression"
    if exp.owner:
    	return [v for inp in exp.owner.inputs for v in find_vars(inp, var_type)]
    else:
        return [exp] if isinstance(exp, var_type) else []

find_vars(g, T.TensorVariable) # [<TensorType(int32, scalar)>, <TensorType(int32, scalar)>] x and y
```

This means we could minimize a cost function using gradient descent like this:

```python
def minimize_cost(inputs, cost, lr): # simplified, should accept all params of theano.function(...)
	return theano.function(inputs, cost, updates=sgd(lr, cost, *find_vars(cost)))
```

Adding regularization is left as an exercise for the reader (hint: usually only the weight matrices are included in the regularization term, not the bias vectors).
