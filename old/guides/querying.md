+++
title = "Querying"
[menu.main]
  parent = "Reference Guides"
  pre = "<i class='fa fa-file-text-o'></i>"
+++

## Creating a Query

Morphia provides `Query<T>` class to build a query and map the results back to instances of your entity classes and attempts
 to provide as much type safety and validation as possible.  To create the `Query`, we invoke the following code:

```java
Query<Product> query = datastore.find(Product.class);
```

`find()` returns an instance of `Query` which we can use to build a query.

### `filter()`

The most significant method `filter(Filter...)`.  This method takes a number of filters to apply to the query being built.  The
 filters are added to any existing, previously defined filters so you needn't add them all at once.  There are dozens of filters
  predefined in Morphia and can be found in the `dev.morphia.query.experimental.filters` package.  
  
{{% notice note %}}
The package is currently `experimental`.  This done to signify that this API is a new one and might change based on user feedback
prior to a final release.  It is expected that the API will be largely the same in the final release and you are highly encouraged
to try it out before then.  If you encounter and bugs or usability issues, please file an 
[issue](https://github.com/MorphiaOrg/morphia/issues)
{{% /notice %}}

The filters can be accessed via the [Filters]({{< apiref "dev/morphia/query/experimental/filters/Filters" >}}) class.  The method names
 largely match the operation name you would use querying via the mongo shell so this should help you translate queries in to Morphia's
  API.  For example, to query for products whose prices is greater than or equal to 1000, you would write this:

```java
query.filter(Filters.gte("price", 1000));
```

This will append the new criteria to any existing criteria already defined.  You can define as many filters in one call as you'd like or
 you may choose to append them in smaller groups based on whatever query building logic your application might have.

## Complex Queries

Of course, queries are usually more complex than single field comparisons.  Morphia offers both `and()` and `or()` to build up more
complex queries.  An `and` query might look something like this:

```java
q.filter(and(
    eq("width", 10), 
    eq("height", 1)));
```

An `or` clause looks exactly the same except for using `or()` instead of `and()`, of course.  The default is to "and" filter criteria
 together so if all you need is an `and` clause, you don't need an explicit call to `and()`:

```java
datastore.find(UserLocation.class)
    .filter(
        lt("x", 5),
        gt("y", 4),
        gt("z", 10));
```

This generates an implicit `and` across the field comparisons.

## Other Query Options

There is more to querying than simply filtering against different document values.  Listed below are some of the options for modifying
the query results in different ways.

### Projections

[Projections]({{< docsref "tutorial/project-fields-from-query-results/" >}}) allow you to return only a subset of the fields in a
document.  This is useful when you need to only return a smaller view of a larger object.  Borrowing from the [unit tests]({{< srcref
"morphia/src/test/java/dev/morphia/TestQuery.java" >}}), this is an example of this feature in action:

```java
ContainsRenamedFields user = new ContainsRenamedFields("Frank", "Zappa");
datastore.save(user);

ContainsRenamedFields found = datastore
                                  .find(ContainsRenamedFields.class)
                                  .iterator(new FindOptions()
                                               .projection().include("first_name")
                                               .limit(1))
                                  .tryNext();
assertNotNull(found.firstName);
assertNull(found.lastName);
```

As you can see here, we're saving this entity with a first and last name but our query only returns the first name (and the `_id` value) in
 the returned instance of our type.  It's also worth noting that this project works with both the mapped document field name
 `"first_name"` and the Java field name `"firstName"`.

{{% notice warning %}}
While projections can be a nice performance win in some cases, it's important to note that this object can not be safely saved back to
 MongoDB.  Any fields in the existing document in the database that are missing from the entity will be removed if this entity is
  saved. For example, in the example above if `found` is saved back to MongoDB, the `last_name` field that currently exists in the database
  for this entity will be removed.  To save such instances back consider using [`Datastore#merge(T)`]({{< apiref "dev/morphia/Datastore#merge-T-" >}})
{{% /notice %}}

### Limiting and Skipping

Pagination of query results is often done as a combination of skips and limits.  Morphia offers `FindOptions.limit(int)` and 
`FindOptions.offset(int)` for these cases.  An example of these methods in action would look like this:

```java
datastore.createQuery(Person.class)
    .iterator(new FindOptions()
	    .offset(1)
	    .limit(10))
```

This query will skip the first element and take up to the next 10 items found by the query.  There's a caveat to using skip/limit for
pagination, however.  See the [skip]({{< docsref "reference/method/cursor.skip" >}}) documentation for more detail.

### Ordering

Ordering the results of a query is done via [`FindOptions.sort(Sort...)`](/javadoc/dev/morphia/query/Query.html#order-java.lang.String-) 
and a handful of other overloads. For example, to sort by `age` (youngest to oldest) and then `income` (highest to lowest), you would
 use this:

```java
getDs().find(User.class)
       .iterator(new FindOptions()
                    .sort(ascending("age"), descending("income"))
                    .limit(1))
       .tryNext();
```

### Tailable Cursors

If you have a [capped collection]({{< docsref "core/capped-collections/" >}}) it's possible to "tail" a query so that when new documents
are added to the collection that match your query, they'll be returned by the [tailable cursor]({{< docsref
"reference/glossary/#term-tailable-cursor" >}}).  An example of this feature in action can be found in the [unit tests]({{< srcref
"morphia/src/test/java/dev/morphia/TestQuery.java">}}) in the `testTailableCursors()` test:

```java
getMorphia().map(CappedPic.class);
getDs().ensureCaps();                                                          // #1
final Query<CappedPic> query = getDs().createQuery(CappedPic.class);
final List<CappedPic> found = new ArrayList<>();

final MorphiaCursor<CappedPic> tail = query.iterator(new FindOptions()
                                                   .cursorType(CursorType.Tailable));
while(found.size() < 10) {
	found.add(tail.next());                                                    // #2
}
```
There are two things to note about this code sample:

1.  This tells Morphia to make sure that any entity [configured]({{< ref "/guides/annotations#entity" >}}) to use a capped
 collection has its collection created correctly.  If the collection already exists and is not capped, you will have to manually 
 [update]({{< docsref "core/capped-collections/#convert-a-collection-to-capped" >}}) your collection to be a capped collection.
1.  Since this `Iterator` is backed by a tailable cursor, `hasNext()` and `next()` will block until a new item is found.  In this
version of the unit test, we tail the cursor waiting to pull out objects until we have 10 of them and then proceed with the rest of the
application.

## Deleting

Queries are also used to delete documents from the database as well.  Using 
[`Query#delete()`]({{< apiref "dev/morphia/query/Query#delete()" >}}), we can delete documents matching the query.  The default operation
 will only delete the first matching document.  However, you can opt to delete all matches by passing in the appropriate options:
 
```java
datastore
    .find(Hotel.class)
    .filter(gt("stars", 100))
    .delete(new DeleteOptions()
                     .multi(true));  
```
