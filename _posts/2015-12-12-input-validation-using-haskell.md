---
layout: post
title:  "Input validation using Haskell"
shortdescription: >
    I describe a way to implement input validation, functional style, using Haskell.
tags: haskell functional-programming
---
I was hacking on a small hobby project of mine in Scala, and the need to perform input validation arose. I wanted to keep it as "functional programming" as possible, by writing expressive code that is also modular and easily integrated and reused. While I was searching around on the internet for hints, an idea stuck in my head: to view an input validation as a function from the type of input to a validation result. In this blog post, I will try to explain how I developed this idea, to handle multiple constraints on a certain input, and how to combine single constraints in a clean way. Haskell will be the language of my choice, since I find it very expressive and well-fitting for this particular problem.

Let's start out with the basics: to define what the result of a validation is. While it is tempting to think that we can simply use `Bool`s -- either the input was valid (`True`) or it was invalid (`False`) -- we want to keep the code expressive, so let's define our own data type `ValidationResult`:

    data ValidationResult = Valid | Invalid

As we can see, this is just an aliased `Bool`, but `Valid`/`Invalid` is more expressive and intuitive than `True`/`False`.

Let's move forward to the next logical step: define what a validation is. An intuitive definition is to take some kind of input and tell whether it is `Valid` or `Invalid`. That translates very naturally to type `Validator a` that takes an argument of type `a` and returns a `ValidationResult`.

    type Validator a = a -> ValidationResult

Notice that I used the word "validator" instead of "validation". I think it is more intuitive to think that something that validates input is a _validator_, since it _performs_ the validation.
