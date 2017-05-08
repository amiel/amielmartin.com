---
layout: post
title: "How Ember Data Loads Async Relationships: Part 1"
date: 2017-05-05 16:02:46 -0800
date_formatted: May 5, 2017
comments: true
published: true
categories:
  - ember.js
  - ember-data
---

<!-- TODO: ember version badge 2.13.1 -->

Ember Data has two main ways to load async relationship data through adapters. It is not always obvious when or why Ember Data calls the adapter hooks that it does, or doesn't.

In this part, we'll explore how Ember Data response to a few different common scenarios. In later posts we'll look at some less-straight-forward examples.

<!--More-->

## Some Housekeeping

Before we get started, let's talk about scope. All examples here will be in `JSONAPI`. In most cases, this will translate pretty easily for users of `RESTAdapter` or `ActiveModelAdapter`.

### Examples

For all of the following examples, we'll be using the simple blog with posts and comments example. This is an easy relationship to use without having to explain any domain concepts.

There is example project at [github.com/amiel/ember-data-relationships-examples](https://github.com/amiel/ember-data-relationships-examples/tree/part-1), if you'd like to follow along with working examples. Many examples are trimmed for brevity and the full source can be easily found by clicking on the filename.

Examples of `JSONAPI` data are hard-coded in [the adapters](https://github.com/amiel/ember-data-relationships-examples/tree/part-1/app/adapters).

Here are the models we'll be working with:

#### [`app/models/post.js`](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/models/post.js)

```javascript
export default DS.Model.extend({
  title: attr('string'),
  body: attr('string'),
  comments: hasMany(),
});
```

#### [`app/models/comment.js`](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/models/comment.js)

```javascript
export default DS.Model.extend({
  post: belongsTo(),
  message: attr('string'),
});
```

## `links`

`JSONAPI` allows for specifying that a [relationship should be loaded via a url](http://jsonapi.org/format/#document-resource-object-related-resource-links) specified by the server.

Let's see how this works with our first example: Post #1.

#### [Post #1 data](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/post.js#L13-L25)

```json
{
  "id": 1,
  "type": "post",
  "attributes": {
    "title": "This is post #1",
    "body": "This post's comments relationship has a links section",
  },
  "relationships": {
    "comments": {
      "links": { "related": "/posts/1/comments" },
    },
  },
}
```

In this example, because the `comments` key under `relationships` matches the name of the `comments` relationship defined in our post model, Ember Data will use the link provided to load data for that relationship. The [default](https://emberjs.com/api/data/classes/DS.JSONAPIAdapter.html#method_findHasMany) [implementation](https://github.com/emberjs/data/blob/v2.13.1/addon/adapters/rest.js#L641-L693)  adds any url prefix configuration to the provided url (such as the host) and fires off an ajax request to the provided link.

Therefore, accessing this relationship would cause the following ajax request:

```javascript
post.get('comments');
// GET /posts/1/comments
```

However, it is possible to override this behavior by defining `findHasMany` in the parent model's adapter:

#### [`app/adapters/post.js`](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/post.js#L51)

```javascript
findHasMany(store, snapshot, link, relationship) {
  // Here, snapshot is the post snapshot,
  // link === "/posts/1/comments"
  // and relationship holds metadata about the relationship, such as
  // relationship.type === 'comment'
  // relationship.kind === 'hasMany'

  // A very simple implementation (using ember-ajax) could be:
  return this.get('ajax').request(link);
},
```

Using links is arguably the simplest way to load relationships with Ember Data if your server supports it. It is also an extremely useful adapter hook to get around some limitations in Ember Data as we will see in a later post in this series.

## Resource Linkage

It is also possible to include the ids for each object in a relationship. `JSONAPI` calls this [`Resource Linkage`](http://jsonapi.org/format/#document-resource-object-linkage).

> Resource linkage in a compound document allows a client to link together all of the included resource objects without having to GET any URLs via links.
>
> -- http://jsonapi.org/format/#document-resource-object-linkage

Let's see how this works with our next example: Post #2.

Unlike Post #1, Post #2 has "data" instead of "links" in the "comments" section. Here's what it looks like:

#### [Post #2 data](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/post.js#L27-L43)

```json
{
  "id": 2,
  "type": "post",
  "attributes": {
    "title": "This is post #2",
    "body": "This post's comments relationship has a data section",
  },
  "relationships": {
    "comments": {
      "data": [
        { "id": 21, "type": "comment" },
        { "id": 22, "type": "comment" },
        { "id": 23, "type": "comment" },
      ],
    },
  },
}
```

Notice that each comment is a ["Resource Identifier Object"](http://jsonapi.org/format/#document-resource-identifier-objects), meaning that it has an `id` and a `type`, but not `attributes`.

With this post, Ember Data will load each comment through the comments adapter by calling its `findRecord` hook.

Therefore, accessing this relationship would cause the following ajax requests:

```javascript
post.get('comments');
// GET /comments/21
// GET /comments/22
// GET /comments/23
```

As before, this bahavior can also be configured. This time by overriding the `findRecord` adapter hook in the comments adapter.

#### [`app/adapters/post.js`](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/comment.js#L5)

```javascript
findRecord(store, type, id, snapshot) {
  // Here, type is an object representing the model class.
  // type.modelName === 'comment'
  // this hook will get called three times in each call, each with an
  // appropritate id and snapshot
  // id === snapshot.id
},
```

Preloading the ids for relationships like this is nice when it makes sense for the interface to show placeholders for each item in the relationship and load details later.

