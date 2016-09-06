Title: Writing queries for Django models with JSONField
Date:2016-09-06
Tags: django, python, jsonfield
Category: django
Authors: José San Gil


I've been writing a lot of python and Django lately and decided to write about the Postgres' `jsonb` type, Django and JSONField.

One interesting feature introduced in Django 1.9 is the `JSONField`. This field adds support for `jsonb`, a data type introduced in [Postgres 9.4](https://www.postgresql.org/docs/9.4/static/release-9-4.html). In case you haven't read/hear about it before, `jsonb` allows you to store JSON data (it actually stores a binary representation) in a table column. It was already possible to store JSON objects as text, but the `jsonb` type enforces data to comply with the JSON format rules and of course, it gives you SQL syntax to make queries over the JSON's attributes.


Creating queries with JSONField seems to be easy according to the [Django documentation](https://docs.djangoproject.com/en/1.9/ref/contrib/postgres/fields/#jsonfield), but once you move into more complex situations, where queries require relations and aggregate functions, things could easily get frustrating.

Let's say we have a table named `users_stats`, that stores daily interactions completed by the users of your application. Each record of the table has three columns: an `id` column that works as incremental primary key (PK), a date `timestamp` column named `date` and a `jsonb` column named `data`. A JSON stored in the `data` column looks like this:


```json
 {
   "clicked_links": 50,
   "visited_pages": 12,
   "viewed_products": 130,
   "sold_products": 8
 }
```

Let's support you want the daily stats summary in a period of time. The following query will retrieve that summary using Django Queryset:


### Using Django ORM

```python
 UserStats.objects.filter(date__range(since, until))
     .annotate(l=RawSQL("((data->>'clicked_links')::int)", (0,)),
               p=RawSQL("((data->>'visited_pages')::int)", (0,)),
               v=RawSQL("((data->>'viewed_products')::int)", (0,)),
               s=RawSQL("((data->>'sold_products')::int)", (0,))) \
     .aggregate(clicked=Sum('l'),
                visited=Sum('p'),
                viewed=Sum('v'),
                sold=Sum('s'))
```

Unfortunately, the query above feels more like a hack than a solution; it's harder to understand than the produced SQL and you have to use `RawSQL` expressions to define each of the aggregate functions (annotations).

The equivalent SQL for this query would be:

```SQL
 SELECT date, SUM((data->>'clicked_links')::int), SUM((data->>'visited_pages')::int),
              SUM((data->>'viewed_products')::int), SUM((data->>'sold_products')::int)
 FROM user_stats
 WHERE date BETWEEN 'YYYY-MM-DD' AND 'Y1Y1Y1-M1M1M1-D1D1'
 GROUP BY date;

```

Using the above query with Django is straightforward; we just need a Django connection object to execute the query:

```python
 from django.db import connection


 query = """SELECT date, SUM((data->>'clicked_links')::int), 
                         SUM((data->>'visited_pages')::int),
                         SUM((data->>'viewed_products')::int),
                         SUM((data->>'sold_products')::int)
            FROM user_stats
            WHERE date BETWEEN 'YYYY-MM-DD' AND 'Y1Y1Y1-M1M1M1-D1D1'
            GROUP BY date;"""

 cursor = connection.cursor()
 cursor.execute(query)
 return cursor.fetchall()
```

Now, let's move that boilerplate code into a function:

### Using SQL

```python
 from django.db import connection


 def execute_raw_query(query):
     """Execute a RawQuery in the Database"""
     cursor = connection.cursor()
     cursor.execute(query)
     return cursor.fetchall()


 query = """SELECT date, SUM((data->>'clicked_links')::int),
                         SUM((data->>'visited_pages')::int),
                         SUM((data->>'viewed_products')::int),
                         SUM((data->>'sold_products')::int)
            FROM user_stats
            WHERE date BETWEEN 'YYYY-MM-DD' AND 'Y1Y1Y1-M1M1M1-D1D1'
            GROUP BY date;"""

 rows = execute_raw_query(query)
```

Finally, we should create a custom manager for the `user_stats` model and add a function that calculates the daily summary properly:

```python
 # myapp/models.py
 from django.db import models
 from myapp.utils import execute_raw_query

 class UserStatsManager(models.Manager):

     def get_daily_user_stats(self, since, to):
         """Calculates user stats summary

         :param since: retrieves stats from this date
         :param to: retrieves stats until this date
         """

         # formatting dates to iso 8601
         since_str = date.strftime('%Y-%m-%d')
         to_str = date.strftime('%Y-%m-%d')

         query = """SELECT date, SUM((data->>'clicked_links')::int),
                                 SUM((data->>'visited_pages')::int),
                                 SUM((data->>'viewed_products')::int),
                                 SUM((data->>'sold_products')::int)
                    FROM user_stats
                    WHERE date BETWEEN '{}' AND '{}'
                    GROUP BY date;""".format(since_str, to_str)

         rows = execute_raw_query(query)
         return rows


  class UserStats(models.Model):
      ...

      objects = UserStatsManager()
```

Both queries do not return Django model objects, but a list of dictionaries containing the date and the aggregation parameters.

## Conclusion

This post doesn't intent to declare that raw queries are the best choice while working with `JsonField`, but show that while support of Postgres' `jsonb` in Django is new, and there's still a lot of room for improvement, you can write complex SQL queries using SQL (as usual). If you find yourself struggling to create a query with JsonField through Django ORM or event worse, you hesitate about using JsonField, because you'll require harder queries than those examples at the Django docs, stop losing time and write some SQL.


#### References
I spent half a hour looking for references on how to create aggregation queries using Django ORM, and found this Stackoverflow [question](http://stackoverflow.com/questions/34325096/how-to-aggregate-min-max-etc-over-django-jsonfield-data) that seems to be the more accurate Django ORM's approach.
