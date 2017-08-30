---
layout: post
title: "How Ember Data Loads Async Relationships: Part 2"
date: 2017-05-17 16:14:44 -0800
date_formatted: May 17, 2017
comments: true
categories:
  - ember.js
  - ember-data
  - series-ember-data-async-relationships
---

<span class="embadge" data-start="2.12.0"><span>

If you haven't already read [Part 1][], I recommend doing that now, as this continues right where we left off.

[Part 1][] explored how Ember Data responds to a few common scenarios. In Part 2, we will look at some less straightforward examples. Then, in a later post, we will examine limitations of Ember Data.

[Part 1]: /blog/2017/05/05/how-ember-data-loads-relationships-part-1/

<!--More-->

## Links _and_ Data with IDs

In [Part 1][], we talked about what happens when data is loaded via relationship `links`, when a relationships `ids` are already loaded, and what happens when neither are present.

So, what happens if _both_ `links` and `ids` are present?

#### [Post #4 data][post-4]

Below we can see that the blog post contains both related `links` and `ids` in `data`.

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

In this case, Ember Data will prefer the `data` and call [`findRecord` in the comment adapter][]. This way, any records in the relationship that have already been loaded won't need to be loaded again. We will look at how reloading relationships works in the next couple of sections.

Anyway, accessing the comments relationship on post #4 will load the comment in `data` and not use the related link:

```javascript
post.get('comments').mapBy('message') // => ["Comment 41 was loaded via findRecord in the comment adapter"]
```

Note that if the post data gets reloaded and only has a `links` section, it will correctly [set `hasLoaded` to false][] so that the next attempt to load the relationship will use the link.

## Reloading `links`

Speaking of subsequent loads of relationship data, let's look at how Ember Data deals with reloading relationships. There are many possible scenarios, so let's arbitrarily start where we just were: an updated link.

Let's assume we have the previous post ([post #4][post-4]) loaded, and when we reload the blog post, we just get a `link`, like this:

```json
{
  "id": 4,
  "type": "post",
  "attributes": {
    "title": "This is blog post #4",
    "body": "This blog post has been updated",
  },
  "relationships": {
    "comments": {
      "links": {
        "related": { "href": "/posts/4/comments" },
      },
    },
  },
}
```

What happens with this example when we try to load the comments relationship?

Because the [link hasn't changed][link-changed-check], it will not cause `hasLoaded` to be [set to false][set-hasLoaded-false]. So accessing the comments relationship after reloading the blog post will continue to use the already loaded data.

However, if the value of the related link changes, it will reload the relationship. For example, if reloading the post returns a link with a different url:

#### [Updated post #4 data][post-4-updated]

```json
{
  "id": 4,
  "type": "post",
  "attributes": {
    "title": "This is blog post #4",
    "body": "This blog post has been updated and has a new related link",
  },
  "relationships": {
    "comments": {
      "links": {
        "related": { "href": "/posts/4/comments?1" },
      },
    },
  },
}
```

Then accessing the relationship will trigger reloading with the `link`:

```javascript
post.get('comments').mapBy('message')
// => ["Comment 41 was loaded via findHasMany in the post adapter", "Comment 42 was loaded via findHasMany in the post adapter"]
```

## Reloading `data`

Ok, let's say we reload the post and now there's more data. For this example, we'll go back to [post #2][post-2], which has three comments in the `data` section. When reloaded, two comments were added and one was deleted.

#### [Updated post #2 data][post-2-updated]

```json
{
  "id": 2,
  "type": "post",
  "attributes": {
    "title": "This is blog post #2",
    "body": "This blog post's comments relationship has a data section, and has been updated with new comments",
  },
  "relationships": {
    "comments": {
      "data": [
        { "id": 21, "type": "comment" },
        { "id": 23, "type": "comment" },
        { "id": 24, "type": "comment" },
        { "id": 25, "type": "comment" },
      ],
    },
  },
}
```

Just like before, the comments are loaded through the [comment adapter's `findRecord` hook][], but this time the only comments loaded are those that _haven't already been loaded_.

We can verify this by adding a [logging statement to the comment adapter][]. After reloading the post, we can see the correct comments.

> Comment 21 was loaded via findRecord in the comment adapter
> Comment 23 was loaded via findRecord in the comment adapter
> Comment 24 was loaded via findRecord in the comment adapter
> Comment 25 was loaded via findRecord in the comment adapter

And in the developer console we see that the [comment adapter's `findRecord` hook][] was only called with the new records:

```text
<ember-data-relationships-examples@adapter:comment::ember361> findRecord for comment 24
<ember-data-relationships-examples@adapter:comment::ember361> findRecord for comment 25
```

## Summary

Depending on the contents of the `relationships` section, ember-data will call different hooks in your adapters to load relationship data. The following table summarizes which hooks will be called in each situation.

<table>
  <thead>
    <tr>
      <th colspan="2" rowspan="2" class="empty-corner"></th>
      <th colspan="2">
        <code>data.@each.id</code>
      </th>
    </tr>
    <tr>
      <th>not present</th>
      <th>present</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2">
        <code>link.related</code>
      </th>

      <th>not present</th>

      <td>nothing</td>
      <td><code>adapter:comment findRecord</code></td>
    </tr>

    <tr>
      <th>present</th>

      <td><code>adapter:post findHasMany</code></td>
      <td><code>adapter:comment findRecord</code></td>
    </tr>
  </tbody>
</table>

## Up Next

In [Part 3][], we'll look at how to load relationships that do not have existing links or data.


[post-4]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/post.js#L54-L71
[post-2]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/post.js#L27-L43
[has-many-state-get-records]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L213
[`findRecord` in the comment adapter]: https://github.com/amiel/ember-data-relationships-examples/blob/part-1/app/adapters/comment.js#L5
[`hasMany` macro]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/has-many.js#L146-L148
[`has-many` relationship state]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js
[calls `getRecords`]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/has-many.js#L147
[relationship state objects]: https://github.com/emberjs/data/tree/v2.13.1/addon/-private/system/relationships/state
[async setting]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L66
[related `link`]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L71
[actual records]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L60
[relationship-state-hasData]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L73
[relationship-state-hasLoaded]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L74
[getRecords-link-check]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L218
[hasLoaded-check]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L219
[setHasLoaded-in-push]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L397
[call-to-findRecords]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L220
[call-to-findLink]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/has-many.js#L222
[set `hasLoaded` to false]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L399
[link-changed-check]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L377
[set-hasLoaded-false]: https://github.com/emberjs/data/blob/v2.13.1/addon/-private/system/relationships/state/relationship.js#L399
[cache-busting query param]: https://stackoverflow.com/questions/9692665/cache-busting-via-params
[post-4-updated]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/post.js#L80-L94
[post-2-updated]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/post.js#L99-L116
[comment adapter's `findRecord` hook]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/comment.js#L10
[logging statement to the comment adapter]: https://github.com/amiel/ember-data-relationships-examples/blob/part-2/app/adapters/comment.js#L11
[Part 3]: /blog/2017/07/17/how-ember-data-loads-async-relationships-part-3/
