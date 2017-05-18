---
layout: post
title: "How Ember Data Loads Async Relationships: Part 2"
date: 2017-05-17 16:14:44 -0800
date_formatted: May 17, 2017
comments: true
categories:
  - ember.js
  - ember-data
---

<iframe width="178" height="24" style="border:0px" src="https://mixonic.github.io/ember-community-versions/2017/05/17/how-ember-data-loads-async-relationships-part-2.html"></iframe>

If you haven't already read [Part 1](http://www.amielmartin.com/blog/2017/05/05/how-ember-data-loads-relationships-part-1/), I recommend doing that now, as this continues right where we left off.

In [Part 1](http://www.amielmartin.com/blog/2017/05/05/how-ember-data-loads-relationships-part-1/), we explored how Ember Data responds to a few different common scenarios. In Part 2, we'll look at some less-straight-forward examples. In a later posts we'll look at how to get around some limitations of Ember Data.

<!--More-->

## Links _and_ Data with ids

In [Part 1](http://www.amielmartin.com/blog/2017/05/05/how-ember-data-loads-relationships-part-1/), we talked about what happens when data is loaded via relationship `links`, when a relationships `ids` are already loaded, and what happens when neither are present.

So, what happens if there are _both_ `links` and `ids`?

#### [Post #4 data](https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/post.js#L54-L71)

This blog post has both related `links` and `ids` in `data`.

```json
{
  "id": 4,
  "type": "post",
  "attributes": {
    "title": "This is blog post #4",
    "body": "This post has mixed links and data",
  },
  "relationships": {
    "comments": {
      "links": {
        "related": { "href": "/posts/4/comments" },
      },
      "data": [
        { "id": 41, "type": "comment" },
      ],
    },
  },
}
```

In this case, Ember Data will prefer the `data` and call [`findRecord` in the comment adapter](https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/comment.js#L5). To understand why, let's look at the codepath for loading this relationship. This [`hasMany` macro](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/has-many.js#L146) defines a computed property, that loads an the [`has-many` relationship state](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js) and [`calls getRecords`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/has-many.js#L147) on it.

This is where we get to the meat of the logic. [`getRecords`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L213) checks if there is [a related link](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L218), which, in the case of blog post #4, there is. Then, it checks if the [relationship `hasLoaded`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L219). What does that mean? I don't know, but we can find [where it is set in `push`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L397).

It looks like, since there is a `data` section in our relationship, [`findRecords`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L220) is called instead of [`findLink`](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L222).

Note, however, that if the post data gets reloaded and only has a `links` section, it will correctly [set `hasLoaded` to false](https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L399) so that the next attempt to load the relationship will use the link.


