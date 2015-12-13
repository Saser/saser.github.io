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

### Defining a single requirement

Let's start out with the basics: to define what a single requirement is, and how it can be validated for some input to see whether it was met or not.

We think about what a requirement is. In perhaps its most basic form, a requirement is a rule which is either followed or not. To understand a requirement, we need some kind of description for it. We also need a way to see whether the requirement is met or not. This leads us to think that a requirement for input of type `a` can be said to be a description given as a `String` and a _validator_ that checks whether the input meets the requirement.

{% highlight haskell %}
    type Requirement a = (String, Validator a)
{% endhighlight %}

We have not yet defined what a `Validator` is. In my opinion, a _validator_ is something that _validates_ some input and tells whether it was valid or invalid. This naturally translates to a function: given some input of type `a`, tell whether it was valid or not.

{% highlight haskell %}
    type Validator a = a -> Bool
{% endhighlight %}

However, since we want to keep our code expressive, returning a `Bool` does not feel quite right. It is more intuitive to think about input being `Valid` or `Invalid`, rather than think that the input being valid was `True` or `False`. We define a new data type `ValidationResult` which, more or less, is an alias for `Bool`.

{% highlight haskell %}
    data ValidationResult = Valid | Invalid
{% endhighlight %}

We change our `Validator` type to return a `ValidationResult` instead of a `Bool`.

{% highlight haskell %}
    type Validator a = a -> ValidationResult
{% endhighlight %}

### Examples of single requirements

In order to keep our sanity in check, and not try to think completely generic and abstract, let's define our requirements from the motivating problem as `Requirement`s.

{% highlight haskell %}
    firstCharNotNumber :: Requirement String
    firstCharNotNumber = (description, validator)
        where
            description = "The input should not begin with a number"

            numbers = ['1' .. '9']
            isNumber c = c `elem` numbers
            validator input = if isNumber (head input) then Invalid else Valid

    thirdCharIsUnderscore :: Requirement String
    thirdCharIsUnderscore = (description, validator)
        where
            description = "The third character should be an underscore"

            validator input = if input !! 2 == '_' then Valid else Invalid

    lastCharIsUnderscore :: Requirement String
    lastCharIsUnderscore = (description, validator)
        where
            description = "The last character should be an underscore"

            validator input = if (last input) == '_' then Valid else Invalid
{% endhighlight %}

Having all these `if` statements in the code gets quite ugly, so let's refactor it a bit. We define a new function `validIf` that simply takes a `Bool` and returns `Valid` if that `Bool` was `True`, and otherwise returns `Invalid`. We also define a new function `invalidIf` in a similar manner.

{% highlight haskell %}
    validIf :: Bool -> ValidationResult
    validIf True = Valid
    validIf False = Invalid

    invalidIf :: Bool -> ValidationResult
    invalidIf = validIf . not
{% endhighlight %}

Now the code for the three requirements looks a little bit cleaner, and quite a lot more expressive.

{% highlight haskell %}
    firstCharNotNumber :: Requirement String
    firstCharNotNumber = (description, validator)
        where
            description = "The input should not begin with a number"

            numbers = ['1' .. '9']
            isNumber c = c `elem` numbers
            validator input = invalidIf $ isNumber (head input)

    thirdCharIsUnderscore :: Requirement String
    thirdCharIsUnderscore = (description, validator)
        where
            description = "The third character should be an underscore"

            validator input = validIf $ input !! 2 == '_'

    lastCharIsUnderscore :: Requirement String
    lastCharIsUnderscore = (description, validator)
        where
            description = "The last character should be an underscore"

            validator input = validIf $ (last input) == '_'
{% endhighlight %}

A nice property of the `validIf`/`invalidIf` functions is that if you already have a `a -> Bool` function that validates your input, you can simply use function composition to create a `Validator a` from it. For example, we can use the `even` function to create a `Requirement Integer` for an `Integer` to be an even number.

{% highlight haskell %}
    evenNumber :: Requirement Integer
    evenNumber = (description, validator)
        where
            description = "The number should be even"

            validator = validIf . even
{% endhighlight %}

In my opinion, this code is as expressive and easy to understand, as the examples above.

### Checking a single requirement

Now that we have defined what a requirement is, we need some generic way to check whether a requirement was met. Let's define a function `validate` that takes a `Requirement a` and an input of type `a`, and returns the `ValidationResult` for the input.

{% highlight haskell %}
    validate :: Requirement a -> a -> ValidationResult
    validate (desc, validator) input = validator input
{% endhighlight %}

Simple enough.

### Combining multiple single requirements

Having a single requirement is usually not enough to validate some input. Now that we have an expressive way to define single requirements, we want some way to combine them and check whether some given inputs meets all of them. To do that I decided to create a new type called `FullRequirements a`, which is simply a list of `Requirement`s.

{% highlight haskell %}
    type FullRequirements a = [Requirement a]
{% endhighlight %}

Now it feels natural to have a function that can check some input against some `FullRequirements`. For that I create a new function `validateAll` that takes a `FullRequirements a`, an input of type `a` and returns a list of `ValidationResult`s.

{% highlight haskell %}
    validateAll :: FullRequirements a -> a -> [ValidationResult]
    validateAll fullReq input = map (\req -> validate req input) fullReq
{% endhighlight %}

There is a fairly big flaw in this design, however: we only return the validation results, without any information about which validations were performed, or which validations returned `Valid` and which returned `Invalid`. Imagine that the return value of this function was what the API returned in the motivating example above. The user could know how many requirements were met, but not which requirements were fulfilled and which were not! That would be even more frustrating. We need some way to associate a validation result with a description of the requirement.

### Making the validation user friendly

Fortunately, solving the abovementioned problem is fairly simple. Instead of returning a simple list of `ValidationResult`s, we return a list of tuples, coupling the description of each validation to its validation result. We only need to modify the type signature of the function and the mapped function.

{% highlight haskell %}
    validateAll :: FullRequirements a -> a -> [(String, ValidationResult)]
    validateAll fullReq input = map (\req@(desc, _) -> (desc, validate req input)) fullReq
{% endhighlight %}

In my opinion this code is quite a bit less expressive and readable than the rest of the code we have written. I think that the `validate` function is unnecessary unless developing interactively and running GHCi, so I decided to change it into a function called `validateInput` that takes an input of type `a` and a `Requirement a`, and gives a tuple with the description of the `Requirement` and its `ValidationResult` for the input.

{% highlight haskell %}
    validateInput :: a -> Requirement a -> (String, ValidationResult)
    validateInput input (desc, validator) = (desc, validator input)
{% endhighlight %}

Now we can replace the lambda function in `validateAll` with a partial application of `validateInput`.

{% highlight haskell %}
    validateAll :: FullRequirements a -> a -> [(String, ValidationResult)]
    validateAll fullReq input = map (validateInput input) fullReq
{% endhighlight %}

Now the code looks a bit more expressive and is easier to understand.
