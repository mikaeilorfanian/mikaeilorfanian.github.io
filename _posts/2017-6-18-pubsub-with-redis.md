---
layout: post
title: Simplify Your App by Using the PubSub Pattern, Part 1 Intro to PubSub Using Python and Redis
published: true
---
_note 1_ Due to the complexity of the topic of PubSub and its implementation, this tutorial will be a multi-part series of articles.  
_note 2_ We'll use technologies like Python and Redis, but everything discussed here is actually technology agnostic.   
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


An `event`(left side of the arrow) triggers some `action`(right side of the arrow) and sometimes an `action` triggers more `event`s.  
In addition, 
- Some `action`s need to be handled asynchronously. For example, we don't want to make a user wait while we send an email.  
- Some `action`s are to be handled immediately and some are scheduled to happen in future.
- Some aspects of the behavior of the system should be configurable by non-developers. 
- Business wants to experiment with new events and actions without investing much time and money.
- Scaling the application should not require a total rewrite or redesign of the application.
- We - the devs - want to write high quality code. We want to have automated tests and readable code that just works!  


## Intro to PubSub
Pubsub is an architecture used for designing complex systems characterized by requirements similar to the ones above. If you adhere to the design guidelines of PubSub, you'd end up with software that is much easier to reason about, maintain, and scale.  
The PubSub architecture is made up of 3 components:  
- _Publishers_ send or trigger events. They don't care where those events go and what happens to them.  
- _The Hub_ (also called a broker) routes `event`s; it tells events where to go.  
- _Subscribers_ act on `event`s; they take `action`s.  They don't care where `event`s come from. A subscriber tells the hub that it's interested in one or more `topic`s. When an event with that `topic` occurs, the broker notifies the subscriber.  


_Note:_ a `topic` is like a tag for an `event`. An `event` can have one or more `topic`s. Also, a subscriber can subscribe to one or more `topic`s.  
The above 3 components effectively divide our application into 3 self-contained modules or components which allows for multiple implementations of the PubSub architecture.  
First, we'll go through one implementation and discuss its strengths and weaknesses. In the next article, we'll go through another implementation that addresses the weaknesses of the first implementation.  
## All-Components-Within-One-Application Implementation
We can code our web framework, publisher, hub, and subscriber within the same application. Those who are familiar with Django signals have already used this implementation of PubSub. For example, you can subscribe to signals - Django `signal`s are similar to PubSub `topic`s- like `pre_save` or `post_save` if you want to trigger some action each time a row in your database is modified.  
The picture below shows how these 3 compoenents interact with each other:  
![pubsub first implementation](/images/pubsub1.png "PubSub First Implementation")  
Let's start with an `event` publisher. The `NewUserCreated` object is an `event` publisher that hides the details of how we send event details to the `hub`.  
```python
# publish.py
from . import hub # implemention comes later

class NewUserCreated: # event class
    topics = {'new_user_created'} # event can have multiple topics

    @classmethod
    def publish(cls, user_id): 
        hub(cls.topics, user_id)

# how events are triggered in another module
from .publish import NewUserCreated

def signup_handler(username, email, password):
    user = make_new_user(username, email, password)
    NewUserCreated.publish(user.id)
    do_some_other_things()
```
When a new user signs up, we send the `new_user_created` topic and event details(`user_id`) to the `hub`.   
The `hub` is an instance of the `PubSubHub` class:  
```python
# hub.py
from collections import defaultdict

class PubSubHub:

    def __init__(self):
        self.topic_subscriber_mapper = defaultdict(set) #1

    def subscribe_to(self, *topics): # a decorator that takes topics as parameters
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
_#1_ The `hub` has a mapping of `topic`s to `subscriber`s. A `subscriber` can register interest in one or more `topic`s using the `subscribe_to` method of the `PubSubHub` class.  
_#2_ The `subscribe_to` method is a [decorator](https://stackoverflow.com/questions/5929107/decorators-with-parameters) that takes one or more `topic`s as its argument. In Python, a decorator takes a function, does something to it, and returns it. In our case, the `subscribe_to` method takes one or more `topic`s and the function(`subscriber`) that is interested in those `topic`s and populates the `topic` to `subscriber` maps in the `hub`. You will see how this decorator is used later on.  
_#3_ This line creates a mapping from a `topic` to an interested `subscriber`.  
_#4_ This special "dunder" method makes an instance of the `PubSubHub` class callabe. That's why we can send events to the hub using `hub(cls.topics, auction_id)` defined as part of the `NewUserCreated` class implemented above.  
_#5_ We get the `topic`s of an event from the event publisher. An event can have more than one `topic`. This loop ensures that we call subscribers interested in each `topic`.  
_#6_ We call each subscriber with parameters we received from the `event` publisher.  
Now, let's look at a subscriber:
```python
# subscribe.py
import emailer

from . import hub

@router.subscribe_to('new_user_created') #1
def email_new_user(user_id):
    user = get_user(user_id)
    context = {
        'subject': 'Welcome!',
        'body': 'Hello{}! Thanks for signing up.'.format(user.first_name),
    }
    emailer.send(user.email, context)
```
_#1_ We're using the `subscribe_to` decorator on top of our `subscriber` function. `subscribe_to` creates a mapping from the `new_user_created` topic to the `email_new_user` function.  
_Note_: The code above doesn't run as is because we haven't created a `router` object yet. Also, we need to trigger the decorators somehow. To see a full impolementation, go to [this repo].  
## Analysis
This implementation is simple and many frameworks are implemented this way. However, our requirements mention that some tasks need to ben run asyncronously and that scalability should not require a rewrite of the whole application.     
### Async
There are two reasons why we want asynchrony:
#### 1
Some tasks like sending an email take a long time to finish and users don't like waiting. We could use an async web framework to enable async in our application, but that would only work if all of our long-running tasks are IO-bound. Plus, the most popular Python web frameworks like Django and Flask don't have async built-in. In short, we don't want to learn and use a totally new framework which doesn't even guarantee to address all the async issues.   
#### 2
Even if we were okay with waiting for long-running tasks to finish, there's still a big reason why we want to have asynchrony.  
Did you notice that if anything in the above implementation breaks every component of the PubSub architecture is affected?   Let's say we have 3 subscribers for the `new_user_created` topic. The `hub` tries to notify the first subscriber, but an exception is thrown. This means that none of the 3 subscribers get notified.  
One way of fixing this would be to catch the exceptions. Since we have 3 major components and a wide range of subscribers, the number of exceptions we'd have to hanlde will be big. This can affect performance and reduce code quality.   
### Scalability
To scale this implementation, we'd have to scale the whole application(add more servers running the same application). As a result, we'd get more publishers, hubs, and subscribers in addition to the main web application. This is a waste of resources because it's rarely the case that we need to scale those 4 components at once. And, sometimes scaling everything can be dangerous. For example, if our subscribers have too many events to process, then adding more publishers would make matters worse.    

## How to Fix These Issues
We can fix the weaknesses of the first implementation by running each PubSub component in a separate application or process. However, this would require the introduction of task queues and buffers to our architecture. In the next part of this series, we'll use tools like Redis, RQ, rqworker, and rqscheduler to do just that.  
If you'd like to be notified when we publish next article in the series, fill out [this form(redirects to our newsletter signup page)]().  

> _Article by Mikaeil Orfanian_
