---
layout: post
title: "How Ember Data Loads Async Relationships: Part 3"
date: 2017-07-17 08:53:32 -0800
date_formatted: July 17, 2017
comments: true
categories:
  - ember.js
  - ember-data
  - series-ember-data-async-relationships
---

<span class="embadge" data-start="2.12.0"><span>

If you haven't already read [Part 1][] and [Part 2][], I recommend doing that now, as this continues right where we left off.

[Part 1][] explored how Ember Data responds to a few common scenarios. [Part 2][] discussed some less straightforward examples. [Part 3] will examine how to load relationships when the API does not provide data or links.

[Part 1]: /blog/2017/05/05/how-ember-data-loads-relationships-part-1/
[Part 2]: /blog/2017/05/17/how-ember-data-loads-async-relationships-part-2/

<!--More-->

Let's revisit a scenario from [Part 1][]. [Post #3][] doesn't have a `relationships` section and therefore does not have any `data` or `links`.

[Post #3]: https://github.com/amiel/ember-data-relationships-examples/blob/part-3/app/adapters/post.js#L45-L52

```json
{
  "id": 3,
  "type": "post",
  "attributes": {
    "title": "This is blog post #3",
    "body": "This post's has no comments",
  },
}
```

When attempting to load a relationship with this post, Ember Data will not call any adapter hooks.

```js
post.get('comments');
// => []
```

In this post we will explore a few different ways to deal with this situation.

## 1. Update your API to include `data` or `links`

Ok, this is kind of a cop-out. Really though, if you have control over the API, it might as well follow the [JSON:API spec for fetching relationships][].

However, if you don't have the ability to change the API, or the way your data is structured just needs to load relationships without knowing a url ahead of time or based on user data, read on.

[JSON:API spec for fetching relationships]: http://jsonapi.org/format/#fetching-relationships

## 2. Manipulate the relationships `links` or `data`

Assuming we can't change our API, and without a `relationships` section in the payload Ember Data doesn't call any of our adapter hooks, we'll have to make a `relationships` section ourselves. This could be done in the adapter or the serializer. Let's do it in the serializer since the adapter is designed to load data and the serializer is meant to manipulate the loaded data.

Another benefit to putting this logic in the serializer is that we can call `this._super` first to make sure that we are already working with the JSON:API structure.

#### [`app/serializers/post.js`][]

```js
normalize(typeClass, hash) {
  let jsonapi = this._super(...arguments);

  if (!jsonapi.data.relationships.comments) {
    jsonapi.data.relationships.comments = {
      links: { related: `/posts/${hash.id}/comments` },
    };
  }

  return jsonapi;
},
```

This same idea could be done with `data`, although I'm struggling to think of a use-case for this.

One bummer about putting this logic in the post serializer is that usually url configuration happens in the adapter.

[`app/serializers/post.js`]: https://github.com/amiel/ember-data-relationships-examples/blob/part-3/app/serializers/post.js#L4-L14



