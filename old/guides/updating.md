+++
title = "Updating"
[menu.main]
  parent = "Reference Guides"
  pre = "<i class='fa fa-file-text-o'></i>"
+++

Updates in 2.0, are issued off of a `Query` instance in contrast to the [old approach](updates-old.md) of using `Datastore#update()` in
 previous versions.  These update operations are executed on the server without pulling any documents in to your application.  Update
  operations are defined using a set of functions as  defined on 
  [UpdateOperators]({{< apiref "dev/morphia/query/experimental/updates/UpdateOperators" >}})  In our examples, we'll be using the
  following model:
 
```java
@Entity("hotels")
public class Hotel
{
   @Id
   private ObjectId id;

   private String name;
   private int stars;

   @Embedded
   private Address address;

   List<Integer> roomNumbers = new ArrayList<Integer>();

   // ... getters and setters
}

@Embedded
public class Address
{
   private String street;
   private String city;
   private String postalCode;
   private String country;

   // ... getters and setters
}
```

### set()/unset()
To change the name of the hotel, one would use something like this:

```java
datastore
    .find(Hotel.class)
    .update(UpdateOperators.set("name", "Fairmont Chateau Laurier"))
    .execute();
```

The `execute()` can optionally take [UpdateOptions]({{< apiref "dev/morphia/UpdateOptions" >}}) if there are any options you might want
 to apply to your update statement.

Embedded documents are updated the same way.  To change the name of the city in the address, one would use something like this:

```java
datastore
    .find(Hotel.class)
    .update(UpdateOperators.set("address.city", "Ottawa"))
    execute();
```

Values can also be removed from documents as shown below:

```java
datastore
    .find(Hotel.class)
    .update(UpdateOperators.unset("name"))
    execute();
```

After this update, the name of the hotel would be `null` when the entity is loaded.

## Multiple Updates

By default, an update operation will only update the first document matching the query.  This behavior can be modified via the optional 
[UpdateOptions]({{< apiref "dev/morphia/UpdateOptions" >}}) parameter on `execute()`:

```java
datastore
    .find(Hotel.class)
    .inc("stars")
    .execute(new UpdateOptions()
        .multi(true));
```

## Upserts

In some cases, updates are issued against a query that might not match any documents.  In these cases, it's often fine for those updates
 to simply pass with no effect.  In other cases, it's desirable to create an initial document matching the query parameters.  Examples of
  this might include user high scores, e.g.  In cases like this, we have the option to use an upsert:

```java
datastore
    .find(Hotel.class)
    .filter(gt("stars", 100))
    .update()
    .execute(new UpdateOptions()
                     .upsert(true));  

// creates { "_id" : ObjectId("4c60629d2f1200000000161d"), "stars" : 50 }
```

## Checking results

In all this one thing we haven't really looked at is how to verify the results of an update.  The `execute()` method returns an instance of
`com.mongodb.client.result.UpdateResult`.  Using this class, you can get specific numbers from the update operation as well as any
 generated ID as the result of an upsert.
 
## Returning the updated entity

There are times when a document needs to be updated and also fetched from the database.  In the server documentation, this is referred to
 as [`findAndModify`]({{< docsref "reference/method/db.collection.findAndModify/" >}}).  In Morphia, this functionality is exposed
  through the [Query#modify()]({{< apiref "dev/morphia/query/Query#modify(dev.morphia.query.experimental.updates.UpdateOperator, dev.morphia.query.experimental.updates.UpdateOperator...)" >}})
   method.  With this method, you can choose to return the updated entity in either the state before or after the update.  The default is
    to return the entity in the _after_ state.  This can be changed by passing in a `ModifyOptions` reference to the operation:
    
```java
datastore
    .find(Hotel.class)
    .modify(UpdateOperators.set("address.city", "Ottawa"))
    execute(new ModifyOptions()
        .returnDocument(ReturnDocument.BEFORE));
```
