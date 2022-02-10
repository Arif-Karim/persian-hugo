---
title: How I Learnt to Double-check my Variable Types
date: 2022-01-28T05:14:34+00:00
image: "/images/matt.png"
author: Matt Eden
description: Recently, I merged a new feature into our repo to try and improve some
  of the existing functionality in our fuzzy matching algorithm. It caused our CodePipeline
  to fail. I’m here to talk about what caused that failure and how it taught me to
  be a little more careful with my typing. Let’s dive in.
categories: []
tags:
- error
- recursion
- dynamodb
- technology
- aws
- python

---
Hey, I’m Matt - a developer on the [**Secure@Build API**](https://github.customerlabs.com.au/iagcl/secure-build-nlp/). Recently, I merged a [**new feature into our repo**](https://github.customerlabs.com.au/iagcl/secure-build-nlp/pull/84) to try and improve some of the existing functionality in our fuzzy matching algorithm. For various silly reasons, I didn’t fully test it and the reviewers approved it anyway, so when it was merged into master it caused our _CodePipeline_ to fail. That alone is something that you can pull a couple of lessons out of, but I’m not here to talk about testing methodology or good PR practice. No, I’m here to talk about what caused that failure and how it taught me to be a little more careful with my typing. Let’s dive in.

# The Context

Before we look at what the error is, I think it’s best that we add a little bit more context around what lead up to the error. The Secure@Build team works on the self-titled API, which is more publicly known as CyberCheck. CyberCheck is a service which allows users to perform some simple analysis of Day1 security documents, providing automatic checking of such elements as whether architects have been approved a document, if the non-functional requirements have been met, what stakeholders are defined, etc. The way in which the API achieves this functionality is by performing a static analysis on a Confluence document and applying a ‘fuzzy matching’ algorithm. How fuzzy that algorithm actually is might be up for debate at this point, but that’s the general idea. Analyse document; get information. Simple.

# The Error

Now that we have some more context, what actually went wrong? Well, our _CodePipeline_ has a few stages to it; as all good pipelines do. In particular, there is an ‘Int’ stage (which runs unit tests, like the HTML parsing) and an ‘Integration’ stage (which runs integration tests, like connecting to Confluence). In this case, it was the latter that failed; one of our endpoints was returning a 502. In other words, the Lambda function was broken. Completely stuffed. “But why?” I hear you ask. Well, that’s a good question. To figure that out, we have to dive into the logs.

First up, we can go straight from the failed stage in the _CodePipeline_ to the logs for that execution of the stage in _CodeBuild_. This is actually how I found out it was a 502 in the first place. Knowing that a 502 error meant that the Lambda function needed fixing, I venture over to _CloudWatch_ and search for the prod (production) Lambda that acts as our Day1 security document handler. Diving into those logs, I find the _real_ reason that the pipeline failed:

    [ERROR] RecursionError: maximum recursion depth exceeded while calling a Python object
    

Okay, what the heck is a `RecursionError`? Well, the programmers among you will be familiar with the concept of [**recursion**](https://www.cs.utah.edu/\~germain/PPS/Topics/recursion.html#:\~:text=Recursion%20is%20the%20process%20of,Take%20one%20step%20toward%20home.), but for the non-programmers here’s a simple explanation:

> **Recursion** describes the act of a **function calling itself** as a method of achieving iteration.

Now see, even though I had _absolutely no idea_ what had caused the error, I did at least already know what it meant based on the name alone. However, it wasn’t much help when it came to debugging the issue. The stack trace was a much bigger help in that regard. Those told me where the error _started_, and it all came back to this line in our Lambda:

    item = write_to_DB(
            requestor_workload_name,
            requestor_name,
            result)
    

What I didn’t mention earlier when I was outlining the background context was that the result of all of the calls to our API are stored in a _DynamoDB_ database. This is part of an attempt at caching that is still being worked on, but that’s the database which is being written to in the code above. Immediately, I thought to myself ‘oh, it must be the result that is creating the recursion error’; yet I still had no idea exactly _what_ part of the result was causing the error and _why_ it would cause the error. This is where we start taking a closer look at what my aforementioned PR actually changed.

# The Solution

Before my PR was merged, the response that a user would get back from calling the API would contain something like this:

    {
        'Security Architect': False,
        'Cloud Architect': True
    }
    

After my PR was merged, the response would instead contain something like this:

    {
        'Security Architect': {'name': 'John Smith', 'approved': False},
        'Cloud Architect': {'name': 'Jane Doe', 'approved': True}
    }
    

On the surface, it might be not seem that the second response would cause an issue. That’s what I thought, too, so I messed around with the Lambda a bunch trying to find the breaking point; that one place in the code where it makes the transition from _working_ to _not working_. Personally, this tends to manifest as `print` statements. A tried and true method, if I do say so myself. However, I’ll admit it didn’t quite cut it this time. I verified what was being passed into that `write_to_DB` method, I looked at what the method itself called and followed the movement of variables through the function stack all the way until the very last line that _actually_ writes something to the DynamoDB table. There was no doubt about it; some part of the API response was causing the `RecursionError` when it was being written to the table. Now you may think, “Well, didn’t you already know that?” and to some extent you’d be right. Yet it can still be important to isolate the exact line which causes an error, especially when you have a bunch of data manipulation happening between the first method call and the last.

In any case, I was out of options. So, with all the information I now had at my disposal after investigating the code, I typed into Google:

> recursion error when inserting item into dynamo db python lambda

And it brought me to [**this StackOverflow post**](https://stackoverflow.com/questions/65822810/why-do-i-get-recursionerror-only-for-1-case-using-the-same-variable-when-inserti) where someone had asked a question, then found a solution and returned back with an answer. Turns out, the problem was actually very simple:

> \[I\] was creating a dict in which one of the variables was not a str, but rather a beautiful soup string: <class ‘bs4.element.NavigableString’> This is okay for python, but it’s not okay for dynamoDB. Solution was basically to convert all these NavigableString to str type.

Yeah, so you know how there’s a name for the Security/Cloud Architect? That’s the problem. It’s the wrong type of string. To give you some idea of what that looks like in practice, here’s a quick breakdown:

    name = parsed_html.find('p', class_='example-name').string # This returns a NavigableString
    stringified_name = str(name) # This makes it usable with DynamoDB
    

And that is why you should always double-check your variable types, because sometimes it can make the difference between being able to write to a database and getting a ‘RecursionError’ instead. [**Another day, another PR.**](https://github.customerlabs.com.au/iagcl/secure-build-nlp/pull/87)

_Thanks for reading! Check out my_ [**_GitHub_**](https://github.customerlabs.com.au/matt-eden)_, and feel free to reach out to me on Teams/Slack._