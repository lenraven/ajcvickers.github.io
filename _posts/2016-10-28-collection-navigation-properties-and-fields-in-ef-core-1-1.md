---
layout: default
title: "Collection navigation properties and fields in EF Core 1.1"
date: 2016-10-28 10:37
day: 28th
month: October
year: 2016
author: ajcvickers
permalink: 2016/10/28/collection-navigation-properties-and-fields-in-ef-core-1-1/
---

# EF Core 1.1
# Collection navigation properties and fields in EF Core 1.1

There has recently been some confusion about what mappings are supported for collection navigation properties in EF Core. This post is an attempt to clear things up by showing:

<ul>
<li>What types of collection are supported</li>
<li>When the backing field can be used directly</li>
</ul>



<h2>Mapping to auto-properties</h2>

Perhaps the most common way to define a collection navigation property is to use a simple ICollection<T> property:

``` c#
public class Blog
{
    public int Id { get; set; }

    public ICollection<Post> Posts { get; set; }
}
```

EF will create the collection for you, usually as a HashSet<T> when using Include and during fixup, or the application can set it to any ICollection<T> that allows objects to be added.

EF Core 1.1 also allows a collection to be exposed as IEnumerbale<T>. For example:

``` c#
public class Blog
{
    public int Id { get; set; }

    public IEnumerable<Post> Posts { get; set; }
}
```

This means that there is no public surface for adding entities to the collection. The collection itself still needs to be a proper collection with a working Add method, and EF will still create one for you, or the application can create it itself.

There is also no need for a setter as long as the entity always has a collection created so that EF never needs to do it.

``` c#
public class Blog
{
    public int Id { get; set; }

    public IEnumerable<Post> Posts { get; } = new List<Post>();
}
```

<h2>Properties with backing fields</h2>

<h3>Using an Add method</h3>

Taking this one step further, you can expose your IEnumerable<T> navigation property and use a backing field to control other mutations. For example:

``` c#
public class Blog
{
    private readonly List<Post> _posts = new List<Post>();

    public int Id { get; set; }

    public IEnumerable<Post> Posts => _posts;

    public void AddPost(Post post)
    {
        // Do some buisness logic here...
        _posts.Add(post);
    }
}
```

<h3>Using a defensive copy</h3>

In all of the examples above the Posts navigation property returns the actual collection. This means that application code could add entities directly by casting to an ICollection<T>. This can be fixed by making a defensive copy, but this requires that EF always read and write directly to the field because adding entities to the defensive copy obviously won't work. There is currently no way to do this with the fluent API, but it is still easy to do by dropping to core metadata. For example:

``` c#
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    var navigation = modelBuilder.Entity<Blog>().Metadata.FindNavigation(nameof(Blog.Posts));

    navigation.SetPropertyAccessMode(PropertyAccessMode.Field);
}
```

Adding fluent API for this is being tracked by <a href="https://github.com/aspnet/EntityFramework/issues/6674">issue 6674</a>.

Once we tell EF to always use the field we can get something like this:

``` c#
public class Blog
{
    private readonly List<Post> _posts = new List<Post>();

    public int Id { get; set; }

    public IEnumerable<Post> Posts => _posts.ToList();

    public void AddPost(Post post)
    {
        // Do some buisness logic here...
        _posts.Add(post);
    }
}
```

In this example:

<ul>
<li>EF accesses the navigation property directly through the field</li>
<li>Mutation of the collection is controlled by

<ul>
<li>Creating a defensive copy</li>
<li>Using an Add method with business logic</li>
</ul></li>
</ul>

<h2>Summary</h2>

EF Core allows navigation properties to be defined in the traditional way. It also allows navigation properties to be exposed as IEnumerable<T>. Finally, EF Core can make use of backing fields, which allows for full encapsulation of the collection.
