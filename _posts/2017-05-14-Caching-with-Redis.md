---
layout: post
title: Caching with Redis and Django, The Clean/Pythonic Way
published: true
---

In this post, we'd like to discuss how we've used Redis to achieve a 100x improvement in our Django application's performance.

##The Problem  
Our mobile application shows a list of online students to teachers. If a teacher is online, the student can then make a VOIP call to that teacher. We keep ...
