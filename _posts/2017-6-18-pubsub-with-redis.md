---
layout: post
title: Simplify Your App by Using the PubSub Pattern Part 1, Intro to PubSub
published: true
---
In this article, we will 
- learn about the publ-subs architecture 
- discuss why and when it is so useful
- solve a common software development challenge using pub-sub
## The Problem
Let's say we have a website with the following requirements:
- new user signs up --> send welcome email
- new user does something for the first time --> notify sales people
- user is inactive for X days --> notify user
- user deletes his/her account with us --> notify customer support


These *events* trigger some *action* and sometimes an could trigger other events.  
In addition, we would like: 
- Some events to be handled asynchronously. For example, we don't want to make a user wait while we send an email.  
- Some events to be handled immediately and some to be scheduled for future.
- Some aspects of the behavior of the system to be configurable by non-developers. 
- To experiment with new events and actions without investing much time and money.
- To write clean code. That means automated tests for readable code that does what it's supposed to.  


## Intro to PubSub
Pubsub is an architecture used for designing complex systems characterized by requirements similar to the ones above. If you adhere to the design guidelines of PubSub, you'd end up with software that is much easier to reason about, maintain, and scale.  
The PubSub architecture is made of 3 components:
*Publishers*: they just send or trigger events. They don't care what happens to the event.
*The Hub*: also called a broker, it just routes events; it tells events where to go.
*Subscribers*: they act on events.  They don't care where the events come from. A subscriber tells the broker that it's interested in a type of event(*topic*). When that type of event occurs, the broker notifies the subscriber.  
The above 3 components effectively divide our application into 3 self-contained modules or components. In the Python world, we'd be able to implement this design multiple ways.  
## Implementation
### 1. Three Modules Within One Application
Given that we have a web application, using this implementation would mean that the publisher, hub, and subscriber are housed within the same application. Those who are familiar with Django signals have already used this implementation of PubSub. For example, if you want to trigger some action each time a row in your database is modified, then you can subscribe to signals - PubSub `topic`s are called `signal`s in Django - like `pre_save` or `post_save`.  
Here's how a we'd program a publisher:
```python
# publish.py
from . import router # implemention comes later


class NewUserCreated:
    topics = {'new_user_created'} #1 numbered explanations below

    @classmethod
    def publish(cls, user_id): 
        router.accept_event(cls.topics, auction_id)
----------------------------------------
```
#### #1
# router.py
def new_user_handler()
While this implementation is simple it doesn't address the async and scalability requirements.  
We could use async web frameworks to solve long-running task issue(only if the problems are IO-bound), but the most popular web frameworks like Django and Flask don't have async built-in. So, we'll have to solve this issue by using separate worker processes that will do the long-running tasks. This solution would require us to set up a task queue which we will do using RQ in the next article.  
To scale this implementation, we'd have to scale the whole application(add more servers running the same application). This means that we'd get more publishers, hubs, and subscribers in addition to the main web application. This is a waste of resources because it's rarely the case that we'd need to scale those 4 components at once.  

### 2. 
Async is a very interesting topic. In short, it'll all about not wasting time. In our case, we don't want to waste our users' time. They click a button and see an "email sent" message. They don't want to wait for us while we send emails in the background, deal with errors, log stuff, trigger events and handle those events, etc.
A great tool that we really like is Redis. Redis + RQ + rqworker + rqscheduler can become the backbone of such an async system. Publishers send events to the hub which(using rqworker) puts them on a queue in Redis. rqscheduler handles the scheduling of events. Subscribers are also implemented using rqworker.

Let's go through the code above this time using pictures that show how different components interact with each other in a pubsub architecture. 
