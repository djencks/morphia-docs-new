+++
title = "References"
[menu.main]
  parent = "Reference Guides"
  pre = "<i class='fa fa-file-text-o'></i>"
+++

Morphia supports two styles of defining references:  the [`@Reference`]({{< apiref "dev/morphia/annotations/Reference" >}}) annotation and the experimental [`MorphiaReference`]({{< apiref 
"dev/morphia/mapping/experimental/MorphiaReference" >}}).  The annotation based approach is discussed 
[here]({{< ref "guides/annotations#reference" >}}).  This guide will cover the wrapper based approach.

{{% notice note %}}
This API is experimental.  Its implementation and API subject to change based on user feedback.  However, users are encouraged to
 experiment with the API and provide as much feedback as possible both positive and negative as this will likely be the approach used
  going forward.
{{% /notice %}}

An alternative to the traditional annotation-based approach is the [`MorphiaReference`]({{< apiref
 "dev/morphia/mapping/experimental/MorphiaReference" >}}) wrapper.  This type can not be instantiated directly.  Instead a `wrap()` 
 method is provided that will construct the proper type and track the necessary state.  Currently, four different types of values are 
 supported by `wrap()`:  a reference to a single entity, a List of references to entities, a Set, and a Map.  `wrap()` will determine how
  best to handle the type passed and create the appropriate structures internally.  This is how this type might be used in practice:
  
```java
private class Author {
    @Id
    private ObjectId id;
    private String name;
    private MorphiaReference<List<Book>> list;
    private MorphiaReference<Set<Book>> set;
    private MorphiaReference<Map<String, Book>> map;

    public Author() { }

    public List<Book> getList() {
        return list.get();
    }

    public void setList(final List<Book> list) {
        this.list = MorphiaReference.wrap(list);
    }

    public Set<Book> getSet() {
        return set.get();
    }

    public void setSet(final Set<Book> set) {
        this.set = MorphiaReference.wrap(set);
    }

    public Map<String, Book> getMap() {
        return map.get();
    }

    public void setMap(final Map<String, Book> map) {
        this.map = MorphiaReference.wrap(map);
    }
}

private class Book {
    @Id
    private ObjectId id;
    private String name;
    private MorphiaReference<Author> author;

    public Book() { }

    public Author getAuthor() {
        return author.get();
    }

    public void setAuthor(final Author author) {
        this.author = MorphiaReference.wrap(author);
    }
}

```

As you can see we have 3 different references from `Author` to `Book` and one in the opposite direction.  It would also be good to note 
that the public API of those two classes don't expose the `MorphiaReference` externally.  This is, of course, a stylistic choice but is the encouraged approach as it avoids leaking out implementation and mapping details outside of your model.  

`setList()` accepts a `List<Book>` and stores them as references to `Book` instances stored in the collection as defined by 
the mapping metadata for `Book`.  Because these references point to the mapped collection name for the type, we can get away with storing only the `_id` fields for each book.  This gives us data in the database that looks like this:

```javascript
> db.Author.find().pretty()
{
	"_id" : ObjectId("5c3e99276a44c77dfc1b5dbd"),
	"className" : "dev.morphia.mapping.experimental.MorphiaReferenceTest$Author",
	"name" : "Jane Austen",
	"list" : [
		ObjectId("5c3e99276a44c77dfc1b5dbe"),
		ObjectId("5c3e99276a44c77dfc1b5dbf"),
		ObjectId("5c3e99276a44c77dfc1b5dc0"),
		ObjectId("5c3e99276a44c77dfc1b5dc1"),
		ObjectId("5c3e99276a44c77dfc1b5dc2")
	]
}
```

As you can see, we only need to store the ID values because the collection is already known elsewhere.  However, sometimes we need to 
refer to documents stored in different collections.  For example, if the generic type of the reference is a parent interface or class, we sometimes need to store extra information.  For these cases, we store the references as full `DBRef` instances so we can track the appropriate collection for each reference.  Using this version, we get data in the database that looks like this:
  
```javascript
> db.jane.find().pretty()
{
	"_id" : ObjectId("5c3e99c06a44c77e5a9b5701"),
	"className" : "dev.morphia.mapping.experimental.MorphiaReferenceTest$Author",
	"name" : "Jane Austen",
	"list" : [
		DBRef("books", ObjectId("5c3e99c06a44c77e5a9b5702")),
		DBRef("books", ObjectId("5c3e99c16a44c77e5a9b5703")),
		DBRef("books", ObjectId("5c3e99c16a44c77e5a9b5704")),
		DBRef("books", ObjectId("5c3e99c16a44c77e5a9b5705")),
		DBRef("books", ObjectId("5c3e99c16a44c77e5a9b5706"))
	]
}
``` 

In both cases, we have a document field called `list` but as you can see in the second case, we're not storing just the `_id` values but 
`DBRef` instances storing both the collection name, "books" in this case, and `ObjectId` values from the Books.  This lets the wrapper 
properly reconstitute these references when you're ready to use them.

{{% notice tip %}}
Before we go too much further, it's important to point that, regardless of the type of the references, they are fetched lazily.  So if 
you multiple fields with referenced entities, they will not be fetched until you call `get()` on the `MorphiaReference`.  If the type is 
a `Collection` or a `Map`, all the referenced entities are fetched and loaded via a single query if possible.  This saves on server round trips but does raise the risk of potential `OutOfMemoryError` problems if you load too many objects in to memory this way.
{{% /notice %}}

A `Set` of references will look no different in the database than the `List` does.  However, `Map`s of references are slightly more 
complicated.  A `Map` might look something like this:

```javascript
> db.Author.find().pretty()
{
	"_id" : ObjectId("5c3e9cad6a44c77fa8f38f58"),
	"className" : "dev.morphia.mapping.experimental.MorphiaReferenceTest$Author",
	"name" : "Jane Austen",
	"map" : {
		"Sense and Sensibility " : ObjectId("5c3e9cad6a44c77fa8f38f59"),
		"Pride and Prejudice" : ObjectId("5c3e9cad6a44c77fa8f38f5a"),
		"Mansfield Park" : ObjectId("5c3e9cad6a44c77fa8f38f5b"),
		"Emma" : ObjectId("5c3e9cad6a44c77fa8f38f5c"),
		"Northanger Abbey" : ObjectId("5c3e9cad6a44c77fa8f38f5d")
	}
}
``` 

References to single entities will follow the same pattern with regards to the `_id` values vs `DBRef` entries.

{{% notice info %}}
Currently there is no support for configuring the `ignoreMissing` parameter as there is via the annotation.  The wrapper will silently drop  missing ID values or return null depending on the type of the reference.  Depending on the response to this feature in generalconsideration can be given to adding such functionality in the future.
{{% /notice %}}
