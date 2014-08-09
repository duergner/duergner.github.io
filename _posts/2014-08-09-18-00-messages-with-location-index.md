---
layout: post
title: Playing with Riak 2.0 CRDTs - Part 1 "Messages with location index"
permalink: riak-crdt-part-1-messages-with-location-index
commentIssueId: 1
---

Part 1 of the Riak 2.0 CRDTs will introduce the demo application and go into more detail about the first model class. To show the details of what we built I choose to use a simple Twitter clone example.

The basic data model consists of an **User** who can write a **Message**. Other users can than favourite this messages. All model classes will have a *id* (String), *dateCreated* (Long) and *lastUpdated* (Long) attribute. An **User** will have a *username* (String). A **Message** has at least either a *text* (String) or a *picture* (String), a *location* (Location) and a reference to the **User** who created it; the **Location** will be an embedded Object with *latitude* (Double) and *longitude* (Double) properties. We'll add some additional properties as the application evolves and more features are added.
 
So an example **User** might look like the following in a JSON representation:

```javascript
{
    "id" : "u_1",
    "dateCreated" : 1234,
    "lastUpdated" : 1234,
    "username" : "duergner"
}
```

Out first **Message* might look like the following, again in JSON representation:

```javascript
{
    "id" : "m_1",
    "dateCreated" : 1235,
    "lastUpdated" : 1235,
    "text" : "Welcome to location based messaging",
    "location" : {
        "latitude" : 48.133333,
        "longitude" : 11.566667
    },
    "author" : "u_1"
}
```

We will use the excellent [Spring Framework](http://projects.spring.io/spring-framework/) in combination with [Spring Boot](http://projects.spring.io/spring-boot/) as a base for building our application. So in order to do basic CRUD operations let's create an implementation for the [Spring Data](http://projects.spring.io/spring-data/) *CrudRepository*. You'll find only excerpts from the whole classes below, the full source will be available at [GitHub](https://github.com/duergner/LocationBasedMessaging).

```java
public interface LBMCrudRepository<T extends Base>
        extends CrudRepository<T, String> {
}
```

For the first iteration we'll simple serialize our objects using [Jackson](https://github.com/FasterXML/jackson) via the integrated solution from Riak's Java client. We'll extend this later on using Riak's map CRDT. The basic procedure for storing a value inside Riak is similar for every class, so we build an abstract superclass for our repositories. *exists* and *delete* are pretty straight forward and we'll omit *findAll* for the first version. That only leaves *save* and *findOne* which is implemented as you see below.

```java
@Override
public <S extends T> S save(S entity) {
    BinaryValue oldValue = entity.binaryValue;
    String id = entity.getId();
    StoreValue.Builder builder =
            (null == id ? new StoreValue.Builder(namespace())
                    : new StoreValue.Builder(location(entity)))
                    .withOption(StoreValue.Option.RETURN_BODY,
                            Boolean.TRUE);
    if (null != entity.vClock) {
        builder.withVectorClock(entity.vClock);
    }
    StoreValue storeValue = builder.build();
    try {
        StoreValue.Response storeValueResponse =
                riakClient.execute(storeValue);
        entity = (S) storeValueResponse.getValue(entity.getClass());
        if (storeValueResponse.hasGeneratedKey()) {
            entity.setId(
                    storeValueResponse.getGeneratedKey().toString(CHARSET));
        }
        else {
            entity.setId(id);
        }
        getProxy().addOrUpdateIndices(oldValue,entity,entity.getId());
        entity.vClock = storeValueResponse.getVectorClock();
        entity.binaryValue =
                storeValueResponse.getValue(RiakObject.class).getValue();
        return entity;
    }
    catch (ExecutionException | InterruptedException e) {
        throw new RecoverableDataAccessException(e.getMessage(), e);
    }
}

@Override
public T findOne(String id) {
    FetchValue fetchValue = new FetchValue.Builder(location(id)).build();
    try {
        FetchValue.Response fetchValueResponse =
                riakClient.execute(fetchValue);
        if (!fetchValueResponse.isNotFound()) {
            T result = fetchValueResponse.getValue(clazz);
            result.setId(id);
            result.vClock = fetchValueResponse.getVectorClock();
            result.binaryValue =
                    fetchValueResponse.getValue(RiakObject.class)
                            .getValue();
            return result;
        }
        else {
            return null;
        }
    }
    catch (ExecutionException | InterruptedException e) {
        throw new RecoverableDataAccessException(e.getMessage(), e);
    }
}
```

Whenever the object passed to save already has its *id* property set we construct a *Location* otherwise we pass it on to Riak to create an id by just using the desired *Namespace*. The *vClock* is attached whenever present, i.e. when the object has been read before and this should be an update. After successfully saving the entity we set *id*, *vClock* and *binaryValue* on the result object.

Before returning the object we'll call the *addOrUpdateIndices()* method to add or update any of the indices associated with this object. This is done via getting the Spring proxy class for the instance in order to facilitate the *@Async* invocation.

So let's have a look at the *addOrUpdateIndices()* implementation for our **Message** class.

```java
@Override
public void addOrUpdateIndices(BinaryValue oldValue, Message message,
                               String id) {
    super.addOrUpdateIndices(oldValue, message, id);
    if (null == oldValue) {
        String geoHash =
                GeoHash.encodeHash(message.getLocation().getLatitude(),
                        message.getLocation().getLongitude(), 6);
        addKeyToIndex(INDEX_LOCATION,
                BinaryValue.create(geoHash, CHARSET),
                BinaryValue.create(id, CHARSET));
        addKeyToIndex(INDEX_USER,
                BinaryValue.create(message.getUser(), CHARSET),
                BinaryValue.create(id, CHARSET));
    }
}
```

As we won't let our users update the location of a message later on and as messages cannot be assigned to other users we don't need to do anything on update here. We utilize the *addKeyToIndex()* method from our superclass. This method does a simple *SetUpdate* and add the *id* of this entity to the supplied key in the index, e.g. the [GeoHash](https://en.wikipedia.org/wiki/Geohash) of the location. We use Geohashes of level six here as they offer a good compromise between accuracy and efficiency when searching. 

```java
void addKeyToIndex(BinaryValue indexName, BinaryValue indexKey,
                   BinaryValue indexValue) {
    riakClient.executeAsync(new UpdateSet.Builder(new Location(
            new Namespace(AbstractRiakLBMCrudRepository.SET_BUCKET_TYPE,
                    indexName), indexKey), new SetUpdate().add(indexValue))
            .build());
}
```

Whenever we delete an object, we also need to remove it's *id* from any associated indices. Therefore the *delete()* method will call an *removeIndices()* method similar to the *addOrUpdateIndices()* one which than will again use the following method to remove this entity from a specific index.

```java
    void removeKeyFromIndex(BinaryValue indexName, BinaryValue indexKey,
                            BinaryValue indexValue) {
        try {
            Location location = new Location(
                    new Namespace(AbstractRiakLBMCrudRepository.SET_BUCKET_TYPE,
                            indexName), indexKey);
            FetchSet.Response response = riakClient.execute(
                    new FetchSet.Builder(location)
                            .withOption(FetchDatatype.Option.INCLUDE_CONTEXT,
                                    Boolean.TRUE).build());
            if (null != response.getContext()) {
                riakClient.executeAsync(new UpdateSet.Builder(location,
                        new SetUpdate().remove(indexValue))
                        .withContext(response.getContext()).build());
            }
        }
        catch (ExecutionException | InterruptedException e) {
            logger.warn("Got {} while trying to removeKeyFromIndex: {}",
                    e.getClass().getSimpleName(), e.getMessage());
        }
    }
```

As ~~Riak~~ the **Riak Java Client** (as pointed out by @russelldb) will only allow remove SetUpdate calls when a *Context* is supplied, we need to call a *FetchSet* before to obtain the *Context*.

So and now, last but not least, we'll have a look at search call which will actually use one of the indices we just built. Our *MessageRepository* offers a method to find all *Messages* which are within a set of GeoHashes.
 
```java
@Override
public List<Message> findAllWithinGeoHashes(Set<String> geoHashes) {
    List<Message> result = new ArrayList<>();
    for (String geoHash : geoHashes) {
        try {
            Location location = new Location(new Namespace(
                    AbstractRiakLBMCrudRepository.SET_BUCKET_TYPE,
                    INDEX_LOCATION), BinaryValue.create(geoHash, CHARSET));
            FetchSet.Response response = riakClient
                    .execute(new FetchSet.Builder(location).build());
            for (BinaryValue binaryValue : response.getDatatype().view()) {
                result.add(findOne(binaryValue.toString(CHARSET)));
            }
        }
        catch (ExecutionException | InterruptedException e) {
            logger.warn(
                    "Got {} while trying to fetch all messages with geoHash {}: {}",
                    e.getClass().getSimpleName(), geoHash, e.getMessage());
        }
    }
    return result;
}
```

We simply again do another *FetchSet* call on each and every GeoHash supplied, iterate over the resulting set and call the *findOne()* method as we simply stored the *id* of each message inside our index.

As you might see here there are several ways to improve.

1. We could use *riakClient.executeAsync()* when iterating over the supplied GeoHash set and use e.g. a Promise A+ implementation to merge the results together
2. Without doing any caching this will issue a *FetchValue* call for each and every element in the result set, which will become very slow.
3. We could add an asynchronous version of *findOne()* and use again e.g. Promise A+ to merge all results together in the end

As we're using Spring already and easy solution to *2.* would be to use Spring's [Cache Abstraction Layer](http://docs.spring.io/spring-framework/docs/current/spring-framework-reference/html/cache.html) to cache calls to *findOne()*.

I plan to add another blog post about utilizing Spring Cache Abstraction together with Google's [Guave Cache](https://code.google.com/p/guava-libraries/wiki/CachesExplained) and [Redis](http://redis.io) to build a quite simple distributed caching system sometime soon.

So far for part 1, you'll find the code example at [GitHub](https://github.com/duergner/LocationBasedMessaging); I'll extend this code as the blog series continues and you will find the code specific for this post at the branch *messages-with-location-index*.

Feel free to comment or contact me with any questions, comments or improvements.
 
{% include twitter_share.html %}