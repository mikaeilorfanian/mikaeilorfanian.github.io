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
- _Subscribers_ act on `event`s; they take `action`s.  They don't care where the `event`s come from. A subscriber tells the hub that it's interested in one or more `topic`s. When an event with that `topic` occurs, the broker notifies the subscriber.  
_Note:_ a `topic` is like a tag for an `event`. An `event` can have more than one `topic`. Each subscriber can subscribe to more than one `topic`.  
The above 3 components effectively divide our application into 3 self-contained modules or components. There are multiple ways of implementing the PubSub architecture. In this article, we'll go through one implementation and discuss its strengths and weaknesses. In the next part of this series, we'll discuss a more scalable implementation.  
## All-Components-Within-One-Application Implementation
Our web application, publisher, hub, and subscriber will be housed within the same application. Those who are familiar with Django signals have already used this implementation of PubSub. For example, if you want to trigger some action each time a row in your database is modified, then you can subscribe to signals - PubSub `topic`s are called `signal`s in Django - like `pre_save` or `post_save`.  
Let's start with an `event` publisher. The `NewUserCreated` object is an `event` publisher that hides the details of how we send event details to the `hub`.    
```python
# publish.py
from . import router # implemention comes later

class NewUserCreated: # event class
    topics = {'new_user_created'} # event can have multiple topics

    @classmethod
    def publish(cls, user_id): 
        hub(cls.topics, user_id)

# how events are triggered in another module
from publish import NewUserCreated

def signup_handler(username, email, password):
    user = make_new_user(username, email, password)
    NewUserCreated.publish(user.id)
    do_some_other_things()
```
When a new user signs up, we send the `new_user_created` topic and event details(`user_id`) to the `hub`.   
The `hub` is an instance of the `PubSubHub` class:  
```python
# router.py
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
_#2_ The `subscribe_to` method is a decorator that takes one or more `topic`s as its argument. In Python, a decorator takes a function, does something to it, and returns it. In our case, `subscribe_to` takes one or more `topic`s and the function that is interested in those `topic`s and populates the topic->subscriber maps in the `hub`. You will see how this decorator is used later on.  
_#3_ This line creates a mapping from a `topic` to an interested `subscriber`.  
_#4_ This special "dunder" method makes an instance of the `PubSubHub` class callabe. That's why we can send events to the hub using `hub(cls.topics, auction_id)` defined as part of the `NewUserCreated` class implemented above.  
_#5_ We get the `topic`s of an event from the event publisher.An event can have more than one `topic`. This loop ensures that we call subscribers interested in each `topic`. As you will soon see, each `subscriber` can register interest in more than one `topic`.  
_#6_ We call each subscriber with parameters we received from the `event` publisher.  
Now, let's look at a subscriber:
```python
# subscribe.py
import emailer

from . import router

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
_Note: _The code above doesn't run as is because we haven't created a `router` object yet. Also, we need to trigger the decorators somehow. To see a full impolementation, go to [this repo].
## Let's Picture This
The picture below shows how these 3 compoenents interact with each other:  
![pubsub first implementation](/images/pubsub1.png "PubSub First Implementation")

## Analysis
This implementation doesn't address the async and scalability requirements.  
### Async
We want asynchrony because some tasks like sending an email take a long time to finish and users don't like waiting. An async web framework might solve this problem only if the long-running tasks are IO-bound. Plus, the most popular Python web frameworks like Django and Flask don't have async built-in. These framworks have features that we really like.  
Even if we're okay with waiting for long-running tasks to finish, there's still a big reason why we want to have asynchrony. 
Did you notice that if anything in the above implementation breaks, every component of the PubSub architecture is affected? For example, let's say we have 3 subscribers for the `new_user_created` topic. The `hub` tries to notify the first subscriber, but an exception is thrown. This means that none of the 3 subscribers get notified.  
One way fixing this issue would be to catch exceptions. Since we have 3 major components and a wide range of subscribers, the number of exceptions we'd have to hanlde will be big. This can affect performance and reduce code quality. Plus, there's no guarantee that you've caught all exceptions unless you write really bad and dangerous code.  
### Scalability
To scale this implementation, we'd have to scale the whole application(add more servers running the same application). This means that we'd get more publishers, hubs, and subscribers in addition to the main web application. This is a waste of resources because it's rarely the case that we'd need to scale those 4 components at once. And, sometimes scaling everything can be dangerous. For example, if our subscribers have too many events to process, then adding more publishers would make matters worse.    

## How to Fix These Issues
Most of the weaknesses of the first implementation can be addressed by using separate processes for each component. However, this would require us to add things like task queues and buffers to our architecture. We will implement the second implementation in the next part of this series with the help of tools like Redis, RQ, rqworker, and rqscheduler.  
Stay tuned. If you'd like to be notified when we publish next part in this series, fill out [this form(redirects to our newsletter signup page)]().
