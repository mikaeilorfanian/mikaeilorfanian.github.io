---
layout: post
title: 'Caching with Redis The Pythonic Way'
published: true
---
# Cahcing with Redis The Pythonic Way
In this post, we'd like to discuss how we've used Redis to achieve a 100x improvement in our Django application's performance.

## The Problem  
Our [mobile application](https://play.google.com/store/apps/details?id=com.airschool.student) shows a list of online students to teachers and vice versa. A teacher or student who opens our mobile app is then able to make a VOIP call to an online student or teacher. If we were to use our main persistent database(Postgres) to enable this feature, we'd quickly run into scalability issues. To see why, let me explain in detail what happens every time you log in to a social application(like facebook or our appp):  
- User logs in. The backend has to update the user's status. 
- This user's friends or other in their social circle is able to see the user's status.
- If there are more than 20 or 30 online users to show, we may need to paginate the results because 1)We don't want to transfer huge amounts of info the app at once 2)We know that the user is unlikely to actually go through all the results, so why send them to the user in the first place?
- If we paginate, we'd need to keep track of the results we've already sent to the user.
- We also need to update the online users' status when they go offline.
Phew..., this feature has turned out to be much more complicated than we had orginally thought. Each time a user goes online, we hit our main db a dozen time and we haven't even considered all the other parts of our app that require the use of our db.  

## Intro to Caching
Our main db needs will soon be under too much stress! Enter Redis. Redis is an in-memory no-SQL db which is much much faster than relational databases. So, we will cache users' status in Redis.  
Now, when a user goes online our back-end first checks Redis for a list of online users before making a database call. Most of the time, the calls to the main db won't be necessary because Redis will have what we need.  
When caching anything, you need to think carefully about the follwing questions:  
1. What data structure will hold your data?
2. When do you update your cached data?
3. When do you expire your cached data?
4. What other parts of your code will use this data?

## First Implementation
Django comes with a low-level API to redis, so let's use it:
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
