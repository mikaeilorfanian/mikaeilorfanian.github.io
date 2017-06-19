---
layout: post
title: Simplify Your App by Using the PubSub Pattern: Part 1, Intro to PubSub Using Python and Redis
published: true
---
_note 1_ Due to the complexity of the topic of PubSub and its implementation, this tutorial will be a multi-part series of articles.  
_note 2_ We'll use technologies like Python and Redis, but you everything discussed here is actually technology agnostic.   
In this article, we will 
- learn about the publ-sub architecture 
- discuss why and when it is useful
- solve a common challenge using pub-sub  


## The Problem
Let's say we want to make a web application with the following requirements:
- new user signs up --> send welcome email
- new user does something for the first time --> notify sales people
- user is inactive for X days --> notify user
- user deletes his/her account with us --> notify customer support  


An _event_(left side of the arrow) triggers some _action_(right side of the arrow) and sometimes an `action` triggers more `event`s.  
In addition,: 
- Some `action`s need to be handled asynchronously. For example, we don't want to make a user wait while we send an email.  
- Some `action`s need to be handled immediately and some are scheduled to happen in future.
- Some aspects of the behavior of the system should be configurable by non-developers. 
- Business wants to experiment with new events and actions without investing much time and money.
- Scaling the application should not require a total rewrite or redesign of the application.
- We - the devs- want to write high quality code. We want to have automated tests and readable code that just works!  


## Intro to PubSub
Pubsub is an architecture used for designing complex systems characterized by requirements similar to the ones above. If you adhere to the design guidelines of PubSub, you'd end up with software that is much easier to reason about, maintain, and scale.  
The PubSub architecture is made up of 3 components:
_Publishers_ send or trigger events. They don't care where those events go and what happens to them.
_The Hub_ (also called a broker) routes `event`s; it tells events where to go.
_Subscribers_ act on `event`s; they take `action`s.  They don't care where the `event`s come from. A subscriber tells the broker that it's interested in one or more `topic`s. When an event with that `topic` occurs, the broker notifies the subscriber.  
_Note:_ a `topic` is like a tag for an `event`. An `event` can have more than one `topic`. Each subscriber can subscribe to be notified about more than one `topic`.  
The above 3 components effectively divide our application into 3 self-contained modules or components. There are multiple ways of implementing the PubSub architecture. In this article, we'll go through one implementation and discuss its strengths and weaknesses.  
## All-Components-Within-One-Application Implementation
Our web application, publisher, hub, and subscriber will be housed within the same application. Those who are familiar with Django signals have already used this implementation of PubSub. For example, if you want to trigger some action each time a row in your database is modified, then you can subscribe to signals - PubSub `topic`s are called `signal`s in Django - like `pre_save` or `post_save`.  
Let's start with an `event` publisher:  
```python
# publish.py
from . import router # implemention comes later


class NewUserCreated: # event class
    topics = {'new_user_created'} # event can have multiple topics

    @classmethod
    def publish(cls, user_id): 
        hub(cls.topics, user_id)


# usage in another module
from publish import NewUserCreated


def signup_handler(username, email, password):
    user = make_new_user(username, email, password)
    NewUserCreated.publish(user.id)
    do_some_other_things()
```
When a new user signs up, we send the `new_user_created` topic and event details(`user_id`) to the `hub`. The `NewUserCreated` object is an `event` publisher that hides the details of how we send event details and its `topic` to the `hub`.  
The `hub` is an instance of the `PubSubHub` class:  
```python
# router.py
from collections import defaultdict


class PubSubHub:

    def __init__(self):
        self.topic_subscriber_mapper = defaultdict(set) #1

    def subscribe(self, *topics): # a decorator that takes parameters
        def real_subscribe(subscriber_function): #2
            for topic in topics:
                self.topic_subscriber_mapper[topic].add(subscriber_function) #3
            return subscriber_function

        return real_subscribe


    def __call__(self, topics, **kwargs): #4
        for topic in topics: #5
            for subscriber in self.topic_subscriber_mapper[topic]:
                subscriber(**kwargs) #6
```
#### Explanation
_#1_ The `hub` has a mapping of `topic`s to `subscriber`s. A `subscriber` can register interest in one or more `topic`s using the `subscribe` method of the `PubSubHub` class.  
_#2_ The `subscribe` method is a decorator that takes one or more `topic`s as its argument. You will see how this decorator is used later on.  
_#3_ This line creates a map between a `topic` and an interested `subscriber`.  
_#4_ This special "dunder" method makes an instance of the `PubSubHub` class callabe. That's why we can send events to the hub using `hub(cls.topics, auction_id)` defined as part of the `NewUserCreated` class implemented above.  
_#5_ We get the `topic`s of an event from the event publisher meaning that an event can have more than one `topic`. This loop ensures that we call subscribers interested in each `topic`. As you will soon see, each `subscriber` can register interest in more than one `topic`.  
_#6_ We call the `subscriber` with the parameters we received from the `event` publisher.  
#### Let's Picture This
#### Analysis
This implementation doesn't address the async and scalability requirements. We could resolve this issue by using an async web framework, but the most popular Python web frameworks like Django and Flask don't have async built-in. So, we'll have to solve this issue by using separate worker processes that will do the long-running tasks. This solution would require us to set up a task queue which we will do using RQ in the next article.  
To scale this implementation, we'd have to scale the whole application(add more servers running the same application). This means that we'd get more publishers, hubs, and subscribers in addition to the main web application. This is a waste of resources because it's rarely the case that we'd need to scale those 4 components at once.  

### 2. 
Async is a very interesting topic. In short, it'll all about not wasting time. In our case, we don't want to waste our users' time. They click a button and see an "email sent" message. They don't want to wait for us while we send emails in the background, deal with errors, log stuff, trigger events and handle those events, etc.
A great tool that we really like is Redis. Redis + RQ + rqworker + rqscheduler can become the backbone of such an async system. Publishers send events to the hub which(using rqworker) puts them on a queue in Redis. rqscheduler handles the scheduling of events. Subscribers are also implemented using rqworker.

Let's go through the code above this time using pictures that show how different components interact with each other in a pubsub architecture. 
