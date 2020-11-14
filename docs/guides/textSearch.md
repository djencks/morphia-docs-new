+++
title = "Text Search"
[menu.main]
  parent = "Reference Guides"
  pre = "<i class='fa fa-file-text-o'></i>"
+++

## Text Searching

Morphia also supports MongoDB's text search capabilities.  In order to execute a text search against a collection, the collection must
have a [text index]({{< docsref "/core/index-text/" >}}) defined first.  Using Morphia that definition would look like this:

```java
@Indexes(@Index(fields = @Field(value = "$**", type = IndexType.TEXT)))
public static class Greeting {
    @Id
    private ObjectId id;
    private String value;
    private String language;

    ...
}
```

The `$**` value tells MongoDB to create a text index on all the text fields in a document.  A more targeted index can be created, if
desired, by explicitly listing which fields to index.  Once the index is defined, we can start querying against it like this [test]({{<
srcref "morphia/src/test/java/dev/morphia/query/TestTextSearching.java">}}) does:

```java
mapper.map(Greeting.class);
datastore.ensureIndexes();

datastore.save(new Greeting("good morning", "english"),
    new Greeting("good afternoon", "english"),
    new Greeting("good night", "english"),
    new Greeting("good riddance", "english"),
    new Greeting("guten Morgen", "german"),
    new Greeting("guten Tag", "german")),
    new Greeting("gute Nacht", "german"));

List<Greeting> good = getDs().find(Greeting.class)
                             .filter(text("good"))
                             .execute(new FindOptions()
                                          .sort(ascending("_id")))
                             .toList();
Assert.assertEquals(4, good.size());
```

As you can see here, we create `Greeting` objects for multiple languages.  In our test query, we're looking for occurrences of the word
"good" in any document.  Using the method `text()` found on the `Filters` class, we can then query for all instances of our search term.  We
 created four such documents and our query returns exactly those four.

If you would like to restrict your search to a specific language, that can also be specified as part of the query:

```java
datastore.find(Greeting.class)
         .filter(text("good")
                     .language("english"))
         .execute(new FindOptions()
                      .sort(ascending("_id")))
         .toList();
```

There are some optional parameters in addition to the language you can specify as part of your text search.  Those paramaters are
 documented on the [TextSearchFilter]({{< apiref "dev/morphia/query/experimental/filters/TextSearchFilter" >}}) class.