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

The applicant enters the following string into the input field: `11_fishbones`. That string satisfies only requirement 2 above, while requirements 1 and 3 are not satisfied. When the applicant submits his application the API returns that the input was invalid because it begins with a number. The web application displays the error next to the input field which is being validated. The applicant, who is a stressed person and currently is in hurry, sighs and changes their input to `a1_fishbones`, and resubmits the application, believing they fixed the error.

Now, the input string satisfies requirements 1 and 2, but requirement 3 is not satisfied. The API once again returns that the input was invalid because the last character is not an underscore, and the web application once again displays an annoying little error message. The applicant gets seriously frustrated and begins to feel helpless: how many more times will this cycle continue? Why cannot the web application (which in extension becomes the API) tell them everything that is wrong in one go, instead of having to use a trial-and-error method until the input gets accepted?

It is fairly interesting how this problem, which originates from a problem in the UX of the API, can become a problem of technical design and ultimately implementation using purely functional programming.

## Solution

My solution to the motivating problem was to define what a requirement is, how it is validated, and how to combine small, independent requirements into a set of requirements for a certain input type that can be checked all at once, telling which requirements are met and which are not.

### A Single Requirement

Let's start out with the basics: to define what a single requirement is, and how it can be validated for some input to see whether it was met or not.

We think about what a requirement is. In perhaps its most basic form, a requirement is a rule which is either followed or not. To understand a requirement, we need some kind of description for it. We also need a way to see whether the requirement is met or not. This leads us to think that a requirement for input of type `a` can be said to be a description given as a `String` and a _validator_ that checks whether the input meets the requirement.

    type Requirement a = (String, Validator a)

We have not yet defined what a `Validator` is. In my opinion, a _validator_ is something that _validates_ some input and tells whether it was valid or invalid. This naturally translates to a function: given some input of type `a`, tell whether it was valid or not.

    type Validator a = a -> Bool

However, since we want to keep our code expressive, returning a `Bool` does not feel quite right. It is more intuitive to think about input being `Valid` or `Invalid`, rather than think that the input being valid was `True` or `False`. We define a new data type `ValidationResult` which, more or less, is an alias for `Bool`.

    data ValidationResult = Valid | Invalid

We change our `Validator` type to return a `ValidationResult` instead of a `Bool`.

    type Validator a = a -> ValidationResult
