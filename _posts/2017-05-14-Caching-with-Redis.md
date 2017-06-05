---
layout: post
title: 'Caching with Redis The Pythonic Way'
published: true
---
In this post, we'd like to discuss how we've used Redis to make an infinite improvement in our Django application's performance. By infinite, I mean the difference between an app that crashes under heavy load and an app that doesn't. The difference between an app that works and an app that doesn't work is infinite!  

## The Problem  
Our educational [mobile application](https://play.google.com/store/apps/details?id=com.airschool.student) has a social feature that shows a list of online students to teachers(and vice versa). If we were to use our main persistent database(Postgres) to enable this feature, we'd quickly run into scalability issues. To see why, here's what happens every time you log in to a social application(like facebook or Skype):  
- You log in. The server has to update your status so others can see that you're online.  
- If there are more than 20 or 30 online users to show, we may need to paginate the results because 1)We don't want to transfer huge amounts of info the app at once 2)We know that the user is unlikely to actually go through all the results, so why send them to the user in the first place?  
- If we paginate, we'd need to keep track of the results we've already sent to the user.  
- We also need to update your status when you go offline.  

Phew..., this feature has turned out to be more complicated than we had orginally thought. Each time a user goes online, we hit our main db at least a dozen times. Our server would crash with a couple thousand users. And, we haven't even considered how other features of the app use our database.  

## Caching to the Rescue
When problems such as this one show up, the usual fix is to use a cache. The idea of caching is used everywhere in the world(digital and physical). When I go shopping, I keep my debit card in my pocket. This way, I don't have to reach into my backpack and search for it when I need it. The same idea is by your personal computer's CPU and Operating System to speed things up. We can use this simple idea to save our application.  
### Enter Redis
Redis is an in-memory key-value database which is much faster than relational databases.  
We will cache(save) users' status in a Redis collection. This collection will be a set of online users' IDs(A Redis set is very similar to a Python `set`).
Now, when you open our app
- We put your status into Redis
- We ask Redis for a list of online users 
- Only if Redis doesn't have this list, we make a call to the main db
- But, most of the time, the calls to the main db won't be necessary because Redis will have what we need.  
The idea behind this strategy is exactly the same idea that I use to keep my most frequently used cards right in my pocket.  

### Not So Fast: A Few Things to Consider
When caching anything, you need to think carefully about the follwing questions:  
1. What Redis data structure will hold your data?
2. When do you update your cached data?
3. When do you expire your cached data?
4. What other parts of your code will use this data?
If you don't know why question 1 is important, I suggest you try the oline interactive Redis tutorial.  
In our solution, we decided to gather all online users' info into one collection. Maybe, we can hold each user's info and status in a separate collection? The answer to this question depends on the last 3 questions above!!  

## First Implementation
Django comes with a low-level interface to redis, so let's use it:  
**code**  
As you can see, this solution is not Pythonic at all. In other words, this code smells. Here's why:
1. We're using raw strings as keys to fetch data from Redis.
2. Since we're using raw strings, it's hard to find out if our online users' cache is being used by other modules.
3. When there's a bug related to this cache, we'd have to look for `set` and `get` methods that use raw strings. And our IDE won't be able to help us much.
4. But, what about other Redis operations like `delete`? Damn, we have to search the codebase for those, too :|
5. If someone else looks at this code, they'd have a hard time understanding what this cache represents and what they can do with it or to it.

## Second Implementation
**code**  
This code is much cleaner because we don't use raw strings anymore and we've made use of OOP to create a cache objects that has a very pleasant API. Here are the benefits of this implementation:
1. Since we have a class which encapsulates our cache, all you need to do to know what this cache is and what you do can do with is read the class definition. 
2. To find out who's using our cache, we can use our IDE to easily find all instances of this cache object.
3. We don't have harcoded raw strings trashing our codebase.
4. We can use this OOP pattern to refactor other caches, too. And, if we define some API for all these cache classes, then we and other readers of the code will have a pretty easy time understading our code.

### Is This Refactoring Really Necessary?
Of course! If we spend some more time on our OOP design, we'd eventually end up with an something to Django's ORM or SQLAlchemy. Those frameworks are considered to be among the most important tools developers regularly use.  

## A Summary
Redis can do much more than we've discusses here. Make sure to learn more about what it can do.  
Also, remember that OOP is a software development paradigm that can help you clean up your code. Languages like Java have given OOP a bad rep, but remember that everything in Python is an object.  
Try to break down your features into steps and those steps into db hits. Even if you decide  not to optimize, you'll have a feeling about how different parts of your app use your db. One day, when you are foced to optimize, you'll know where to start.

## Some Code
```python
from django import caches

serialized_online_teachers = {}
online_status_cache = caches['online']

online_teacher_ids = online_status_cache.get('online_teachers')
for teacher_id in online_teacher_ids:
	teacher = User.objects.get(pk=teacher_id)
    serialized_online_teachers.update(teacher.serialize('summary')
```
