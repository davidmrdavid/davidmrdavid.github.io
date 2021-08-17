---
layout: post
title: "Durable Functions for Python just got faster!"
categories: misc
---

<!-- __Note: this is a cross-post from the Durable Functions for Python [announcement](https://techcommunity.microsoft.com/t5/apps-on-azure/why-we-are-so-excited-about-durable-functions-for-python/ba-p/2176099) in the Microsoft Tech Community site.__ -->

__TLDR; we just made Durable Functions for Python way faster. Try it out by installing azure-functions-durable 1.1.0 on PyPI.__

The Durable Functions (DF) programming model aims to make it easy for developers to write stateful workflows that take full advantage of the performance and scaling characteristics of serverless. As such, and since the release of the Durable Functions for Python experience, we’ve been working hard to optimize the performance of DF Python apps to accelerate the parallel processing of tasks, and the execution of concurrent orchestrations. We’re excited to share that we’ve made dramatic improvements to the SDK’s runtime performance, and we can’t wait to get you to try it out!

### How to try it out:
_Install the package_

To try out the new experience, install azure-functions-durable 1.1.0 (or a later version) from PyPI.

This is a non-breaking change release, so your existing applications can benefit from these improvements without any code changes after you upgrade to the latest SDK.

### The performance improvements

The exact performance improvements you will see depend on your workload, your orchestration’s structure and, since we’re dealing with Python, it will especially vary depending on your Python language worker’s [settings](https://docs.microsoft.com/en-us/azure/azure-functions/python-scale-performance-reference).

That said, we can consider for example just a simple fan-out-fan-in orchestration over 5k activities.

```python
## orchestrator function in “Orchestration/__init__.py” 
def orchestrator_function(context: df.DurableOrchestrationContext): 
    activities = [] 
    for i in range(5000): 
        activities.append(context.call_activity('Hello', str(i))) 
    yield context.task_all(activities) 
    return "Done" 

## activity function in “Hello/__init__.py” 
def main(name: str) -> str:  
    return f"Hello {name}!" 
```

In this case, we’ll leave the default Python language worker settings intact, which we generally recommend be fine-tuned. We’ll also leverage the latest durable-extension release. Finally, we’ll run this benchmark on top of the Azure Functions Consumption Plan for Linux. We compare the duration of each orchestration in the graph below.

![](./../../../../../imgs/graph.png)

We see that the SDK in version v1.0.3 takes about 246 minutes to complete, whereas version v1.1.0 takes merely 13 minutes! It’s a dramatic speed-up of about 18x!

Again, your mileage might vary, but we expect that generally all major kinds of workloads will benefit, in runtime performance, from this release.

### The technical story

There’s a bit of a technical story to these changes and, while we don’t have the time and space to get into the nitty-gritty details here, we do want to share some high-level intuitions about what’s changed.

If you’re a Durable Functions user, it hopefully comes as no surprise that DF leverages an [event-sourcing-based](https://docs.microsoft.com/en-us/azure/architecture/patterns/event-sourcing) [replay](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp#reliability) mechanism for fault tolerance. Your orchestration code is replayed multiple times to determine, among other things, what scheduled work is yet to be completed, and what work needs to be scheduled for execution. Given that computationally expensive tasks should be offloaded to activity functions, most of the time spent by an orchestration function will be spent doing this replay-based bookkeeping. This makes the performance of our replay behavior one of the main factors in overall orchestration performance.

During replay, the SDK needs to reconstruct an orchestration’s state at multiple points in the execution. To do this, it needs to reason over an array of “History” events that record things like: “timer scheduled”, “timer fired”, “activity with name X and input Y was scheduled”, “activity with name X and input Y failed”, etc. Using these events, Tasks are determined to have reached a terminal state, or not.

In our old replay algorithm, a single Task could require an exhaustive search over the History to determine if it had finished executing. This meant that we were incurring, in the worst-case scenario, on a runtime complexity of $$O(H*T)$$, where H is the size of the History and T is the number of Tasks.  

Given that the History size is mainly controlled by the number of Tasks, we can simplify this expression to be $$O(T^2)$$, meaning that the time spent on replay increased quadratically in the number of Tasks. This cost isn’t really noticeable in small orchestrations, but it makes a huge difference if your orchestrator makes a lot of Activity calls.

So how can we optimize this procedure? Our approach was to aim at rebuilding an orchestration’s state by processing the History array only once, which we knew we could do by making use of additional metadata in our History events. Our changes can be summarized in two steps.

First, we implemented a lightweight Task state-machine in Python, to facilitate modeling the full lifecycle of DF Tasks. Each Task state machine also monitors the lifecycle of its sub- Tasks, which are needed to implement our WhenAny and WhenAll abstractions. Using these, we can easily propagating task-resolution information across the DF orchestration.

Second, our History array contains metadata that roughly maps to lifecycle updates of Tasks from within the Durable Extension. Now that we have a Task-like state machine in Python, we make use of this metadata to update our Python tasks as we iterate through the History events, allowing us to more efficiently reason about Task states.  

Putting it all together, these changes allow us to make a linear pass over the History events, updating our Task state machines in Python at every relevant event, and using that to more efficiently perform our replay-based bookkeeping and get back to scheduling “real” work.  And there you have it!

### What’s coming

The technical challenges highlighted here are present in other DF implementations as well, particularly in our JavaScript and TypeScript SDKs. Porting these improvements to those SDKs is high in our priority list.

There’s also much work being done behind the scenes to continue improving our out-of-process DF experiences, so be on the lookout for updates like this in the future.

### Keep in touch!

This is a big change so if you encounter any issues, please report them to our GitHub repository here.

Keep in touch with me on Twitter via [@davidjustodavid](https://twitter.com/davidjustodavid)

Other members of Durable Functions team can be found on Twitter via [@cgillum](https://twitter.com/cgillum), [@comcmaho](https://twitter.com/comcmaho), and [@AzureFunctions](https://twitter.com/AzureFunctions).  
