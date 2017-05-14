---
layout: post
title: 'Caching with Redis and Django, The Clean/Pythonic Way'
published: true
---

In this post, we'd like to discuss how we've used Redis to achieve a 100x improvement in our Django application's performance.

## The Problem  
Our [mobile application](https://play.google.com/store/apps/details?id=com.airschool.student) shows a list of online students to teachers. If a teacher is online, the student can then make a VOIP call to that teacher. We keep ...
## The Solution - First Implementation
To relieve the stress on our persistent database, we will cache users' status in Redis. Once this is done, our back-end first checks Redis for a list of online users before making a database call. When caching anything in Redis, you need to think carefully about the follwing questions:  
1. What data structure will hold your data?
2. When do you update your cached data?
3. When do you expire your cached data?
4. What other parts of your code will use this data?

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
