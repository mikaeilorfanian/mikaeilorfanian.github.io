---
layout: post
title: 'Caching with Redis The Clean Way'
published: true
---
In this post, we'd like to discuss how we've used Redis to make an infinite improvement in our Django application's performance. By "infinite", I mean the difference between an application that crashes under heavy load and one that doesn't.

## The Problem  
Our educational [mobile application](https://play.google.com/store/apps/details?id=com.airschool.student) has a social feature that shows a list of online students to teachers and vice versa. Here's the summary of the feature:  
- After you open our app on your phone, our backend server has to update your status so others can see you. They will see your name, a short bio, and your rating.  
- Then, we show you the list of other online users. We may need to paginate the results, because 1)we don't want to transfer huge amounts of data to the app at once 2)we know that the user is unlikely to actually go through all the results. 
- If we paginate, we'd need to keep track of the results you've already seen.  
- Finally, we need to update your status when you go offline.  

Once we lay out the technical details of the feature, we see that each user's status is fetched from the database each time another user goes online. For example, let's say we have 4 users. Each time a user wants to see who else is online, we'd have to check our database. Basically, we're asking our database the same question multiple times. If we were to use our main database(Postgres backend) to enable this feature, it would quickly run out of breath.  

## The Solution: Caching to the Rescue
Problems of this kind are usually solved by using a cache. The idea of caching is often used to improve efficiency both in the digital and physical worlds. For example, while working you cache the most frequently used items by putting them on your desk. Your personal computer's CPU and Operating System also use caching to speed things up.  
To save our main database, we're going to cache users' status, name, and bio by keeping them in memory. As a result, this feature will not use our main database at all.

### Enter Redis
Redis is an in-memory key-value database that can be used for caching purposes. We use Redis extensively because of the Python libraries out there that make things like caching data very simple(more on that later). Now, when you open our mobile app  
- We add your ID to a Redis `set` (which is very similar to a Python `set`).
- We fetch that set from Redis to show you who else is online. 
- We remove your ID from that set when you go offline.  

### Not So Fast: A Few Things to Consider
One reason why Redis is so much faster than relational databases is that it has no idea about the structure of your data. This lack of structure is liberating, but it also has some consequences.  
When caching anything, you need to think carefully about the follwing questions:  
1. What data structure - in our case, what Redis data structure - will hold your data?
2. When do you need to update your cached data?
3. Will this data be used by other features or modules in your application?
If you don't know why question 1 is important, I suggest you try the [interactive online Redis tutorial](http://try.redis.io).  
In our case, we decided to gather all online users' info into one collection. Maybe, we can hold each user's info and status in a separate collection? The answer to this question depends on the last 2 questions.  

### Keeping the Cache Up-to-date
The issue of having accurate data in your cache is by far the most important and challenging thing you'll encounter. Getting bad data quickly is worse than waiting for accurate data to arrive.  
To make sure we have accurate data in Redis, we have to identify sources of discrepancy. There's discrepancy in cached data when that data doesn't accurately reflect what you care about. In our case, discrepancy can be caused when
- user goes online or offline --> we fail to show others that you're online or offline
- user changes their bio --> we show outdated bio to others
- user's rating changes --> we show inaccurate ratings
We know what triggers the first two changes: user opens or closes our application and user updates his/her bio using a form. User's rating is more complicated because it can change when
- another user rates this user
- a previous rating changes or is removed
- our rating algorithm changes
You can now see why it's hard to keep your cached data up-to-date. There are ways to deal with this complexity, but as the "first implementation" below shows, there are also ways to make the situation worse.  

## First Implementation
Note 1: read [this tutorial](https://realpython.com/blog/python/caching-in-django-with-redis/) to learn about how you can install Redis and use it with Django.  
Note 2: the purpose of the pseudo code below is to illustrate relevant concepts. Missing functions, objects, and modules are irrelevant to this discussion.  
Note 3: we won't discuss the caching of ratings and user bio. But, everything we learn from now on applies to caching any kind of data.  
In this first implementation, we'll see how we use the cache when a user opens and closes our mobile app:  
```python
def login_handler(request):
    do_stuff()
    user = request['user'].authenticate()
    cache.set_add('online_users', user.id) # add user's ID to set of online users
    do_other_stuff()
    

def logout_handler(user):
    do_stuff()
    cache.set_remove('online_users', user.id) #remove user's ID from set of online users
    do_other_stuff()
```
### Analysis
The above code has a few problems:  
1. We're using strings as keys to fetch data from Redis. If the key changes, we'd have to make changes everywhere the cache is used. All the `set`, `get`, and `delete` operations will have to be edited.
2. Since we're using strings, it's hard to find where this cache is being used because IDEs won't be able to help us with that.
3. After reading the above code, it's not clear what's modifying our cached data. What is setting it? What is updating it? What if there's a bug and for some reason our cache is not up-to-date? How would you go about finding what's causing that bug?   
Now, let's see how we get a list of online users to show our user:  
```python
def get_online_users():
    online_users_ids = cache.get('online_users')
    if not online_users_ids:
        online_users_ids = User.objects.filter(status='online').values_list('id', flat=True)
        cache.set('online_users' online_users_ids)
    
    return online_users_ids
```
*Note: *In this function, we check whether the list of online users is in Redis. This check is necessary because key-value pairs in Redis have an expiration date(also called `ttl` or `time-to-live`).  
The above code has the same issues as the previous two. For example, this function lives in a module different than the previous two functions. But, it's using the same hardcoded key to access cached data. Also, `get_online_users` is where we set the cache(or where data is cached). This is weird because the function name starts with "get", not "set".  
## Second Implementation
We can solve the above issues by creating a model class(similar to Django model classes) for our cached data. This object would encapsulate all the different ways we access and modify the cached data:
```python
class OnlineUsersCacheManager:
    CACHE_KEY = 'online_users'
        
    @classmethod
    def get_or_set(cls):
        online_users_ids = cache.get(cls.CACHE_KEY)
        if not online_users_ids:
            cls.set_cache()
            online_users_ids = cache.get(cls.CACHE_KEY)
        return online_users_ids

    @classmethod
    def set_cache(cls):
        online_users_ids = User.objects.filter(status='online').values_list('id', flat=True)
        cache.set(cls.CACHE_KEY, online_users_ids)
```
### Analysis
Here's how this implementation is better than the previous one:
1. We don't have harcoded strings trashing our code.
2. To find out what's using or modifying our cache, we can simply use our IDE to find all instances of `OnlineUsersCacheManager`.
3. Since we have a class encapsulating our cache, we can simply read its definition and learn about what it does and what you can do with it.  

## Summary
- Redis is a powerful tool. It's well worth it to get acquainted with it.  
- OOP is a software development paradigm that can help you clean up your code. Languages like Java have given OOP a bad rep, but remember that everything in Python is an object.  
- Try to break down your features into steps and those steps into db hits. Even if you decide  not to optimize, you'll have a feeling about how different parts of your app use your db. One day, when you are foced to optimize, you'll know where to start.

If you'd like to be notified when we publish the next article, sign up using [this form]().
