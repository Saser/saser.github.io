---
layout: post
title:  "Input validation using Haskell"
shortdescription: >
    I describe a way to implement input validation, functional style, using Haskell.
tags: haskell functional-programming
---
I was hacking on a small hobby project of mine in Scala, and the need to perform input validation arose. I wanted to keep it as "functional programming" as possible, by writing expressive code that is also modular and easily integrated and reused. While I was searching around on the internet for hints, an idea stuck in my head: to view an input validation as a function from the type of input to a validation result. In this blog post, I will try to explain how I developed this idea, to handle multiple constraints on a certain input, and how to combine single constraints in a clean way. Haskell will be the language of my choice, since I find it very expressive and well-fitting for this particular problem.

## Motivation

The motivating idea behind my solution is that some kind of API can tell a client exactly what is wrong with the input given to the API.

Let's pretend that I have a REST API that provides some kind of "applying for an account" functionality. There exists a web application that consumes my API, and uses a regular form for users wanting to apply for an account. When applying for an account, the applicant has to fill out a few fields with information. However, these fields each have some fairly specific requirements that all need to be met in order to continue with the application process. Currently the API can tell if the input given to a specific field is valid or invalid, and the reason for it being invalid.

There is a problem with how the API currently works: if the input to a field is invalid for more than one requirement, the API will return that the input was invalid, but will only give the first requirement that was not met. For example: one input field is a regular text input, and the following requirements are set for that text input:

1.  it should not begin with a number
2.  the third character should be an underscore (`_`)
3.  the last character should also be an underscore (`_`)

The applicant enters the following string into the input field: `11_fishbones`. That string satisfies only requirement 2 above, while requirements 1 and 3 are not satisfied. When the applicant submits his application the API returns that the input was invalid because it begins with a number. The applicant, who is a stressed person and currently is in hurry, sighs and changes their input to `a1_fishbones`, and resubmits the application, believing they fixed the error.

Now, the input string satisfies requirements 1 and 2, but requirement 3 is not satisfied. The API once again returns that the input was invalid because the last character is not an underscore. The applicant gets seriously frustrated and begins to feel helpless: how many more times will this cycle continue? Why cannot the API tell them everything that is wrong in one go, instead of having to use a trial-and-error method until the input gets accepted?

## Solution

Let's start out with the basics: to define what the result of a validation is. While it is tempting to think that we can simply use `Bool`s -- either the input was valid (`True`) or it was invalid (`False`) -- we want to keep the code expressive, so let's define our own data type `ValidationResult`:

    data ValidationResult = Valid | Invalid

As we can see, this is just an aliased `Bool`, but `Valid`/`Invalid` is more expressive and intuitive than `True`/`False`.

Let's move forward to the next logical step: define what a validation is. An intuitive definition is to take some kind of input and tell whether it is `Valid` or `Invalid`. That translates very naturally to type `Validator a` that takes an argument of type `a` and returns a `ValidationResult`.

    type Validator a = a -> ValidationResult

Notice that I used the word "validator" instead of "validation". I think it is more intuitive to think that something that validates input is a _validator_, since it _performs_ the validation.
