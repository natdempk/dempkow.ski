+++
date = '2025-02-02T15:05:41-05:00'
draft = false
title = 'Django Prefetch Related Objects: Why Am I Still Seeing N+1 Queries with prefetch_related_objects?'
+++

I've been working on a project that uses Django's ORM heavily and running into a lot of N+1 queries. An N+1 query is a side effect of the "magic" (runtime binding/resolution :roll_eyes:) that the Django ORM enables where accessing fields on a `Model` that correspond to other related models.

This means you can easily write code like this:

```python
for book in Book.objects.all():
    print(book.author)
```

This lets you easily access related objects, here assuming that book and author are modeled as two related objects with a DB-layer relationship between them. This is "desirable" behavior because it makes writing your application code "easier".

The huge downside to being able to easily do this, is that the details of when you make a database query are hidden from you, making it easy to accidentally write a lot of sequential database queries as your application code, and models grow in complexity. These repeated queries are the "N+1 queries" and often result in poorer performance compared to writing a single database query that fetches all the related objects.

Django's solution to this problem is "prefetching". Prefetching to me is a bit of a weird way to describe this, as this is just regular data fetching compared to the default lazy-fetching behavior, but hey. There are a few ways you can do this depending on your input data. The two that are often used together are `prefetch_related_objects` and the `Prefetch(...)` class which offers you more control over how the prefetched-related query is attached to your original objects.

You probably would end up writing some code at some point like this:

```python
books = Book.objects.all()

prefetch_related_objects(
  books, 
  'author',
  Prefetch(
    'fans',
    queryset=Fan.objects.filter(followers__gt=100),
    to_attr='popular_fans'
  )
  'popular_fans__favorite_book',
  'popular_fans__followers',
)

for book in books:
    print(book.author)
    print(book.popular_fans.favorite_book)
    # etc.
```

The idea being that you want to compute some custom field that might be some object with other related objects, and efficientlly prefetch the related objects to that sub-field to avoid N+1 queries. You would imagine here we'd get a justifiable number of queries each fetching a relevant set of many objects:

- `Books`
- `Books.authors`
- `Books.popular_fans`
- `Books.popular_fans.favorite_book`
- `Books.popular_fans.followers`

This is relatively justifiable for what we're doing here which is querying for a core set of data, hydrating a few related objects, and then joining them onto the core object in memory.

The unexpected problem here is that this doesn't work as you might expect. Here the prefetches for `popular_fans__favorite_book` and `popular_fans__followers` will not be included in the original query, and you will end up with some N+1 queries for each of the attributes of `popular_fans` that you explictly tried to prefetch.

To fix this, you instead must bundle the prefetches into the `Prefetch` object like:

```python
prefetch_related_objects(
  books, 
  'author',
  Prefetch(
    'fans',
    'fans__favorite_book',
    'fans__followers',
    queryset=Fan.objects.filter(followers__gt=100),
    to_attr='popular_fans'
  )
)
```

This will result in the queries we expected above, with one bulk query for each of the related object types.

As for exactly why the first approach doesn't work, and why these two aren't just equivalent, I'm not sure. This isn't really documented anywhere I could easily find, and I haven't dug into the source code yet to figure out why exactly this doesn't work.

To me this is a classic example of a place where the "helpful" ORM gets in the way and causes a couple of subtler and harder to debug and understand issues than the core problem it is solving. The better approach is probably to just write explicit SQL and rely only on an object mapper, rather than a full-blown ORM for querying so you can actually keep track of what is going on without subtle performance footguns.
