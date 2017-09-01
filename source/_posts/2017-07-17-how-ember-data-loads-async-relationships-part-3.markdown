---
layout: post
title: "How Ember Data Loads Async Relationships: Part 3"
date: 2017-07-17 08:53:32 -0800
date_formatted: July 17, 2017
comments: true
categories:
  - ember.js
  - ember-data
---

<span class="embadge" data-start="2.12.0"><span>

Greetings! Please read [Part 1][] and [Part 2][] before continuing on.

[Part 1][] explored how Ember Data responds to a few common scenarios. [Part 2][] discussed some less straightforward examples. Part 3 will examine how to load relationships when the API does not provide data or links.

[Part 1]: /blog/2017/05/05/how-ember-data-loads-relationships-part-1/
[Part 2]: /blog/2017/05/17/how-ember-data-loads-async-relationships-part-2/

<!--More-->

Let's revisit a scenario from [Part 1][]. [Blog Post #3][] doesn't have a `relationships` section and therefore does not have any `data` or `links`.

[Blog Post #3]: https://github.com/amiel/ember-data-relationships-examples/blob/part-3/app/adapters/post.js#L47-L54

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

Really though, if you have control over the API, it might as well follow the [JSON:API spec for fetching relationships][].

What if you don't have the ability to change the API?  What if your data is structured such that you need to load relationships a) without knowing a url ahead of time, or b) based on user data?  Let's read on to find a solution!

[JSON:API spec for fetching relationships]: http://jsonapi.org/format/#fetching-relationships

## 2. Manipulate the relationships `links` or `data`

Assuming we can't change our API, and without a `relationships` section in the payload Ember Data doesn't call any of our adapter hooks, we'll have to make a `relationships` section ourselves. This could be done in the adapter or the serializer. Let's do it in the serializer since the adapter is designed to load data and the serializer is meant to manipulate the loaded data.

Another benefit to putting this logic in the serializer is that we can call `this._super` first to make sure that we are already working with the JSON:API structure.

### Hard-code `links` in the Serializer

#### `app/serializers/post.js`

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

### Move Url Building Logic to the Adapter

One bummer about putting this logic in the post serializer is that usually url configuration happens in the adapter. There is likely to be duplication of concerns because there is logic to build urls in the adapter _and_ the serializer.

To get around this, we can inject `links` in the serializer with a [sentinal value][] and build the url in the adapter. That might look something like this:

#### [`app/serializers/post.js`][]

```js
normalize(typeClass, hash) {
  let jsonapi = this._super(...arguments);

  if (!jsonapi.data.relationships.comments) {
    jsonapi.data.relationships.comments = {
      links: { related: 'urlTemplate:comments' },
    };
  }

  return jsonapi;
},
```

#### [`app/adapters/post.js`][]

```js
findHasMany(store, snapshot, link/*, relationship */) {
  let url;

  if (link === 'urlTemplate:comments') {
    url = `/posts/${snapshot.id}/comments`;
  } else {
    url = link;
  }

  return this._super(...arguments);
}
```

This example is hard-coded for one case and it wouldn't be too hard to generalize it to work for any relationship. In fact, this is exactly what [ember-data-url-templates does][].

### Limitations

There are some limitations to this approach. In particular, loading relationships will only work for models that have been run through the serializer. This will not work, for example, with a branch new model created by `store.createRecord`, until that record is saved (and the response from the server passed through `normalize`).

[sentinal value]: https://en.wikipedia.org/wiki/Sentinel_value
[`app/serializers/post.js`]: https://github.com/amiel/ember-data-relationships-examples/blob/part-3/app/serializers/post.js#L4-L14
[`app/adapters/post.js`]: https://github.com/amiel/ember-data-relationships-examples/blob/part-3/app/adapters/post.js#L131-L136
[ember-data-url-templates does]: https://github.com/amiel/ember-data-url-templates/pull/36

## 3. Change Ember Data

Let's review. We are looking at various strategies for loading relationships with Ember Data when the API does not provide `links` or `data` in `relationships`. The first suggestion is to change the API if possible. If it is not possible to change the API, `links` can be injected in the resulting JSON:API structure to mimic an API that does provide these things.

Wouldn't it be nice, though, if Ember Data supported this use-case natively?

The last approach requires Ember Data to change so that supporting this case is a first-class citizen (this has been proposed before). I'm not sure exactly what that would look like. One reason I wrote this blog series is to move the discussion forward. So, what do you think?

[been proposed before]: https://elements.heroku.com/addons/scout

