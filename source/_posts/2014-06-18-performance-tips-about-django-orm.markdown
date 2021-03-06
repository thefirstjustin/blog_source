---
layout: post
title: "Performance Tips about Django ORM"
date: 2014-06-18 14:41
comments: true
categories: ['Django']
tags: ['python', 'performance']
---

Django provides an friendly Object Relational Mapping (ORM) framework. In
several of my data analysis projects, I used Django ORM to process millions of
logcat data generated by hundreds of Android phones. Here are some of the
experiences and tips that helps making the processing just a bit faster. 

<!--more-->


### `DEBUG` Flag

First of all, set `DEBUG` to `False` in `settings.py`. With `DEBUG` as `True`,
Django will keep in memory the DB queries it has run so far, which lead to
memory leak if you have a large batch of importing work.

### Control Transaction Manually

By default, Django will wrap each database operation a separate transaction, and
commit them automatically. Accessing database frequently definitely will slow you
down, especially when all you want to do is just to insert (a large amount of)
data. Django's [transaction][transaction] module provides several functions to let
you control when to commit the transaction. My favorite one is to use
`transaction.commit_on_success` to wrap the function that import data for a
individual device. An addition benefit is, now you know the data importing for
each device either finished completely, or didn't get imported at all. So if
something wrong happens during the importing, or you have to stop it in the
middle for some reason. Next time when you rerun the importing, you won't get
duplicate rows!

[transaction]: https://docs.djangoproject.com/en/dev/topics/db/transactions/


### Bulk Create Rows

When you have lots of data that you want to import into the database, instead of
call each objects `save` function individually, you can store them in a list and
use the object manager's `bulk_create` function. It'll insert the list of
objects into the database "in an efficient manner". Use this technique together
with the `transaction.commite_on_success` mentioned above, the data importing
should be fast enough.


### Iterator

Now all the raw data is imported into database, the next thing you want to do
is probably run second pass of processing, filtering, or whatever. When the data
size is large, it's unlikely that you need to use them again and again. Most of
the time, you just want to iterate through each log line, get some statistical
information, or some simple computation. So after you construct your (crazy)
query set, you want to add an `.iterator()` function after it, so Django knows
you just want to iterate the data once, and will not bother to cache them.
Otherwise, Django will cache the query results, and soon you will find your
system freezes, and the kernel does nothing but swapping...


### Reset Queries And Garbage Collection

Every now and then you can also reset Django queries manually with the
`reset_queries` function, and trigger garbage collection using `gc.collect()`.
They'll help you to further reduce memory usage.

### Resources

[Database access optimization][django_db]

[django_db]: https://docs.djangoproject.com/en/dev/topics/db/optimization/

