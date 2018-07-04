---
layout: post
title: "Django F Expressions & Model-Less Serialization"
author: "Jason Johns"
categories: development
tags: [python,django,models,serialization]
image: city-1.jpg
---


Django Rest Framework and its serializations are an extremely powerful tool to enable creation of API resources. However, there are some situations where the nesting serialization is less than optimal. For example, if you have to construct a dashboard application, you really have two choices: multiple API calls per model to display, or construct a nested serialization of the result set. Both these solutions come with the probable caveat of requiring some transforming logic in your client consumer before you can use it.

The content here came from a project in which I had to construct a spreadsheet grid application using react-data-grid for internal usage. Having all the data serialized in a nested format required logic implemented to transform the API response into something that can be shown in a grid type layout.

A sample db schema:

{% gist 5fd009d887d36561805c4f568d01eaef %}

Here, we have a sample one to many structure where an `Institution` can have many `Catalog`s, each with many `Link` types. If we do a classical nested serializer implementation as described in the [DRF documentation](http://www.django-rest-framework.org/api-guide/relations/#example), then we can have something like

{% gist cc28c586139203c1e40e4400465ad250 %}

where the nesting is top-down from `Institution` to `Link`. A resource endpoint of `/api/institution` returns the following snippet from data generated using factory-boy:

```
{
   "id": 253,
    "name": "Robinson-Henderson",
    "address": "4065 Nicole Lakes Apt. 404",
    "city": "New Jennifer",
    "state": "Georgia",
    "catalog_set": [
        {
            "id": 1002,
            "institution_id": 253,
            "academic_year": "2017-2018",
            "catalog_type": "UG",
            "link_set": [
                {
                    "id": 5001,
                    "media_type": "P",
                    "url": "http://davis.com/",
                    "catalog": 1002
                },
                {
                    "id": 5201,
                    "media_type": "P",
                    "url": "http://smith.com/",
                    "catalog": 1002
                },
                {
                    "id": 5401,
                    "media_type": "P",
                    "url": "http://www.jackson-holland.com/",
                    "catalog": 1002
                },
                {
                    "id": 5601,
                    "media_type": "P",
                    "url": "http://kim.com/",
                    "catalog": 1002
                },
                {
                    "id": 5801,
                    "media_type": "P",
                    "url": "https://vincent.biz/",
                    "catalog": 1002
                }
            ]
        },
        ...
        ...
    ]
}
```

A response formatted this way will require some internal logic to transform to be usable with the grid components. That ends up causing delays in the client because of the nested iteration required. Here, there are three levels, so any transformer method will be O(n³). What if we could move this to the database instead, and work with the correctly formatted response right away?

---

# Enter F Expressions

Django’s models have [F objects](https://docs.djangoproject.com/en/1.11/ref/models/expressions/#django.db.models.F), which represent the value of a model field or annotated column. Instead of doing a database query to pull the value of the field into Python for operation, we can use F objects to do it all in the database. Here, we’ll use this to construct aliases for the fields we want to show.

Since this transformation will occur on the model queryset, the best place for this logic is on a custom model manager. For ease of demonstration, this will be a `LinkManager`

{% gist 4ef49732de46335ac7ac60ba4cc747d1 %}

and then update the `Link` model to use the custom manager. Now, this method will be available with the call Link.objects.get_composite_data().

The way this works is a dict of field names to F objects is defined. Using

    'address': F('catalog__institution__address')

as an example, it is saying Alias this Django model lookup for this institution's address to the field address. It is then used to annotate the queryset, and only the keys from the dict mapping are used to extract out the values from the queryset.

```
In [1]: records = Link.objects.get_composite_data()
In [2]: records[0]
Out[2]:
{'address': '4065 Nicole Lakes Apt. 404',
 'catalog_id': 1002,
 'city': 'New Jennifer',
 'create_date': datetime.date(2017, 11, 18),
 'inst_id': 253,
 'inst_name': 'Robinson-Henderson',
 'link_id': 5001,
 'media_type': 'P',
 'state': 'Georgia',
 'type': 'UG',
 'update_date': datetime.date(2017, 11, 18),
 'url': 'http://davis.com/',
 'year': '2017-2018'
}
```

---

# Serializing

So lets see. We have the data from the database in a flat format. How can we serialize this within DRF to push out to the client as JSON? Enter in model-less serializers! Interestingly enough, a DRF serializer doesn’t have to be bound to a Django model, or any kind of object. Here, I’ve defined a serializer to match up with all the fields of the values list queryset response:

{% gist ad2b3c2f882a75b26e0c2d22ff7c8f71 %}

The downside for my implementation of the db-less model is the serializer needs to define the fields on which are to be serialized, which can make for a somewhat verbose serializer implementation.

---

# Views and Results

I defined an implementation of DRF’s ListAPIView to use the custom model manager method and serializer defined earlier

{% gist c1f0d8e5d12e3e0b72cbe53b9cc8c96d %}

Now, retrieving data from the REST endpoint api/composite returns a result looking like

```
[
 {
  "inst_id": 253,
  "inst_name": "Robinson-Henderson",
  "state": "Georgia",
  "city": "New Jennifer",
  "year": "2017-2018",
  "type": "Undergraduate",
  "url": "http://taylor.com/",
  "media_type": "PDF",
  "create_date": "2017-11-18",
  "update_date": "2017-11-18"
 },
 {
  "inst_id": 254,
  "inst_name": "Davis, Klein and Meza",
  "state": "North Carolina",
  "city": "East Brycemouth",
  "year": "2017-2018",
  "type": "Graduate",
  "url": "https://www.grimes.com/",
  "media_type": "Web",
  "create_date": "2017-11-18",
  "update_date": "2017-11-18"
 },
 ...
 ...
]
```

which is much easier to use in a dashboard view! Best of all, everything is done in the database, and not within Python or Javascript, so performance impact is extremely minimal compared to other solutions. According to django-silk profiling, the data return requires just two queries and four join operations in all.
