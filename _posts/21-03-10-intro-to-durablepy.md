---
layout: post
title: "Durable Functions, a stateful serverless programing model, now available for Python!"
categories: misc
---

__Note: this is a cross-post from the Durable Functions for Python [announcement](https://techcommunity.microsoft.com/t5/apps-on-azure/why-we-are-so-excited-about-durable-functions-for-python/ba-p/2176099) in the Microsoft Tech Community site.__

I'm super excited to announce that Durable Functions for Python is now generally available!
Folks, this has been in the works for some time now, and a lot of effort has gone into it. The rest of the Durable Functions team and I are really thankful to everyone that contributed, and continues to contribute, as well as the community feedback on GitHub. All in all, we look forward to seeing the Durable programming model available for Python devs in the serverless space.

To get started, find the quickstart tutorial [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/quickstart-python-vscode)!
You can also check out our PyPI [package](https://pypi.org/project/azure-functions-durable/) and our GitHub [repo](https://github.com/Azure/azure-functions-durable-python).
Finally, we encourage you to bookmark and review our Python Azure Functions performance tips [here](https://docs.microsoft.com/en-us/azure/azure-functions/python-scale-performance-reference).

If you aren’t very familiar with Durable Functions but are a Python developer interested in serverless, let me get you up to speed! Alternatively, if you’re a programming languages nerd, like me, and are interested in programming models and abstractions for distributed computing and the web more generally, I think you'll find this interesting too.

### The serverless programming trade-off

If you’ve been in the serverless space for any amount of time, then you’re probably familiar with the programming constraints that this space generally imposes on a programmer. In general, serverless functions must be short-lived and stateless. 

From the perspective of a serverless vendor, this makes sense. After all, a serverless app is an instance of a distributed program, so it needs to be resilient to all sorts of partial failures which include unbounded latency (some remote worker is taking too long respond), network partitions / connectivity problems, and many more. And so, if your functions are short-lived and stateless, a serverless vendor trying to recover from this kind of failure can just retry your function again and again until it succeeds.  

You, as the programmer, take on these programming constraints in exchange for the many benefits of going serverless: elastic scaling, paying-per-invocation, and focusing on your business logic instead of on server management and scaling specifics.

### Stateless woes and short-lived blues

In general, these programming constraints aren’t necessarily a big deal. After all, the serverless space is well-known as good fit for generating “glue code” that facilitates the interaction between other larger services; this is often well suited for those coding constraints.

But what happens when you do need state and long-running services? Here’s where we would have to get creative and perhaps take on a burden of a different kind. For example, to bring back state, you might take a dependency on some external storage, like a queue or a DB. Things get hairy quickly as your apps grows in complexity: you might need to add more storage sources, figure out a way to retry operations, a strategy for communicating error states, and will encounter many other difficult ways to shove-in state back to your stateless program. This is all possible to overcome, and folks have done it, but it’s not pretty. Can we do better? 

### Breaking serverless barriers with Durable Functions

Durable Functions is the result of identifying many application requirements that were previously difficult to satisfy in a serverless setting and developing a programming model that made them easy to express.

Let’s take a look at an example, in Python, of course.

Let’s say you’re trying to compose a series of serverless functions into a sequence, turning the output of the previous function into the input to the next. An example of this could be a serverless data analysis pipeline of 3 steps: get some data, process it, and render the summarized results. To make things more interesting, you also want to catch any exceptions in that pipeline and recover from them by calling some clean-up procedure.  

So, how do we specify this with Durable Functions? See below.

```Python
def orchestrate_pipeline(context: df.DurableOrchestrationContext): 
    try: 
        dataset = yield context.call_activity('DownloadData') 
        outputs = yield context.call_activity('Process', dataset) 
        summary = yield context.call_activity('Summarize', outputs) 
        return summary 
    except: 
        yield context.call_activity('CleanUp') 
        Return "Something went wrong" 
```

Let's put the calling conventions and syntax minutia to the side for a moment. The snippet above is simply calling serverless functions in a sequence and composing them by storing their results in variables and passing those variables as the input to the next function in the sequence. Additionally, it wraps that sequence in a try-catch statement and, in the presence of some exception, it calls our clean-up procedure.

The snippet above is deceitfully simple. Remember that this is a serverless program, so each of these functions could be executing on a remote machine, and so normally we’d be required to communicate state across functions via some intermediate storage like a queue or DB. But here, it’s just simple variable-passing and try-catches. If you squint your eyes, it feels like the first naïve implementation you’d come up with if this was running on a single machine. In fact, that’s the point.  

Durable Functions is giving you a framework that abstracts away the minutia of handling state in serverless and doing that for you. It integrates deeply with your programming language of choice such that syntax and idioms like if-statements, try-catches, variable-passing all work and serve as familiar tools of specifying a stateful serverless workflow.

### Diving deeper into Durable Functions  

We’re only scratching the surface here. The Durable Functions programming model introduces a wealth of tools, patterns, concepts, and abstractions to facilitate the development serverless apps. They are documented at length [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=python) so I encourage you to follow-up there if you want to learn more.  

Once you start getting the hang of things, you can dive into more advanced concepts like Entities, which allow you to explicitly read and update shared pieces of state in serverless. These are really convenient if you need to implement a circuit breaker [pattern](https://dev.to/azure/serverless-circuit-breakers-with-durable-entities-3l2f) for your application. You can learn more about them [here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-entities?tabs=python) and [here](http://christophermeiklejohn.com/serverless/2019/05/23/stateful-serverless-database-example.html).

Taking a step back from all these, let’s look at the bigger picture: programming a distributed application. The question of what are the right abstractions when programming for distribution is a question as old as our ability to ping a remote computer. That question has only grown more pressing as more of our applications have gone to the cloud and our business logic has gone serverless. Within this landscape, Durable Functions provides an answer to the challenge of implementing serverless workflows with ease; and that’s why I’m so excited to share this.

### So, why Python?

Durable Functions is currently available in .NET, JavaScript, and most recently in Python as well. Considering the explosion of Data Science-centric use-cases in the Python community, I think serverless is well-suited for powering the kinds of large-scale data gathering and transformation pipelines that are so common in this domain.

Additionally, Python is a very productive high-level language which I think aligns well with the “batteries included” promise of serverless; I think many Python devs will feel right at home with a framework like Durable Functions that abstracts over the challenges of mainstream serverless coding constraints.

Finally, like many others, Python is a language that’s very close to my heart. Like for many other younger programmers, Python is the language I learned to program with and, while in recent years I’ve become quite fond of more obscure PLs (you know who you are), Python is very much still the language that comes to my mind when I think about code. So that’s all to say that I’ve poured my heart into this work and it’s quite meaningful to me to finally have it ready. Even now with the release announcement, we still have many more ideas and projects to help this Python library mature, become more performant, more expressive, and all that good stuff. We hope you enjoy it!

### Learn more and keep in touch!

Keep in touch with me on Twitter via [@davidjustodavid](https://twitter.com/davidjustodavid)
and give [@cgillum](https://twitter.com/cgillum) a follow for more Durable Functions goodness

We’re also quite active on the project’s GitHub issue board so you can always reach out there!
Finally, thank you for reading. If you have any questions and/or want to nerd-out about this, DM me!