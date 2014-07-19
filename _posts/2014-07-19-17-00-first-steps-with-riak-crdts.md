---
layout: post
title: Playing with Riak 2.0 CRDTs - Part 0: "Introduction"
permalink: riak-crdt-part-0
---

Lately I was working on a project building a mobile, location-based messaging solution - to be more precisely building the backend for native Android and iOS clients (as well as a web client version).

Having used Basho's Riak key-value store in several other projects before and really enjoyed it due to it's really simple possibility to scale horizontally I had a deeper look into the upcoming 2.0 release which features CRDT (convergent replicated data types).

What made CRDTs so useful in this project was, that they delegate the responsibility to handle concurrent update conflicts to the database with some simple rules for conflict resolution that totally fit the projects requirements.

We first started out using the **SET** semantics data type for creating a simple index for the location queries. We calculated the GeoHash for every incoming message's location; this string is than used as key for a set semantics bucket and the ID of the newly created message is added to this set. The set conflict resolution rules define that add win or removes, which is fine here as we do a add only (ok we also eventually remove the ID from the set once it is no longer valid, but that's fine also).

Later when we need to display all messages for a specific geographic region we again calculate the geohases for that region, do a lookup in the set semantic bucket for all necessary keys and finally load the single messages again by key from their bucket.

That solution proved to be way faster than using 2i indexes on the geohash strings like the first naive implementation did.

During the next posts I'll dive a little deeper into the architecture of the solution we built by highlighting some of the aspects.

{% include twitter_share.html %}