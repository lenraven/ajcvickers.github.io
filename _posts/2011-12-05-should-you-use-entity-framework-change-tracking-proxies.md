---
layout: default
title: Should you use Entity Framework change-tracking proxies?
date: 2011-12-05 22:42
day: 5th
month: December
year: 2011
author: ajcvickers
permalink: 2011/12/05/should-you-use-entity-framework-change-tracking-proxies/
---

# Entity Framework 4.2
# Should you use Entity Framework change-tracking proxies?

<p>In my last <a href="/2011/12/05/entity-types-supported-by-the-entity-framework/">post</a> I described the different types of entity that EF supports. But which type of entity should you use?</p><p>My opinion, which may be different from others on the EF team, is this:</p>  <ul>   <li>Never use EntityObject or IPOCO; there are no benefits in doing so. </li>    <li>Use POCO entities and make navigation properties virtual so you get lazy loading. </li>    <li>If you're serializing your entities, then consider switching off proxies and forgoing lazy loading since deserializing proxies can be tricky. </li> </ul>  <p>So when should you use change-tracking proxies? To answer that let's take a closer look at the advantages and disadvantages of change-tracking proxies over other types of POCO entities.</p>  <h3>Advantages of change-tracking proxies</h3>  <p>The advantages of change-tracking proxies are quite simple:</p>  <ul>   <li>For tracked entities, changes are detected immediately and fix-up of relationships happens immediately instead of when DetectChanges is called. </li>    <li>There <em>may</em> be performance gains in some scenarios. </li> </ul>  <h3>Disadvantages of change-tracking proxies</h3>  <p>Some of the disadvantages of change-tracking proxies are:</p>  <ul>   <li>The rules that your classes must follow to enable change-tracking proxies are quite strict and restrictive. This limits how you can define your entities and prevents the use of things like private properties or even private setters. The rules are:      <ul>       <li>The class must be public and not sealed. </li>        <li><em>All </em>properties must have public/protected virtual getters and setters. </li>        <li>Collection navigation properties must be declared as ICollection<T>. They cannot be IList<T>, List<T>, HashSet<T>, and so on. </li>     </ul>   </li>    <li>Because the rules are so restrictive it's easy to get something wrong and the result is you won't get a change-tracking proxy. For example, missing a virtual, or making a setter internal. </li>    <li>At runtime, EF sets collection navigation properties to an instance of the EF class EntityCollection<T>. You can't change this, and any collection that you set (for example, in the constructor) will be discarded when EF sets the EntityCollection<T> instance. </li>    <li>Complex properties still require snapshot change-tracking (and hence DetectChanges) if the complex objects are mutated. This is because complex types are not proxied. If you never mutate complex objects (which is, in my opinion, good practice) then they won't require DetectChanges, but by default EF assumes that they might have been mutated. </li>    <li>Entities behave very differently when being tracked by the context than when they are not being tracked. This is because the proxies can't use the context to do fix-up when the entity is not tracked. </li>    <li>You must use DbSet.Create instead of the “new” operator to create instances of the change-tracking proxies, or accept that instances created with “new” will not be change-tracking. Having new entities not be change-tracking proxies can be okay, but you should be aware of it. </li>    <li>Performance <em>may</em> be <em>worse </em>in some scenarios. </li> </ul>  <h3>Performance of change-tracking proxies</h3>  <p>You may have noticed that performance can be both an advantage and a disadvantage of change-tracking proxies. This is because the performance depends on what you do with the entities.</p>  <p>Before getting into some specific performance scenarios its worth noting that for a lot of applications there is no noticeable difference between the performance of change-tracking proxies and entities that use snapshot change tracking. A rule of thumb is that you need to be tracking hundreds or thousands of entities at the same time in a context instance before real-world apps will see a difference. For the vast majority of apps the simpler snapshot change tracking will be just fine.</p>  <p>Change-tracking proxies provide the greatest performance benefit when a lot of entities are being tracked and changes are made to only a few of them. This is because snapshot change tracking needs to scan and compare all entities to detect these few changes whereas with change-tracking proxies each change is detected immediately with no scanning.</p>  <p>Now consider the converse—several changes being made to every tracked entity. In this case the extra cost of recording and reacting to every change when it is made starts to mount up. This “extra cost” exists because setting a simple .NET property is usually extremely fast. This means adding additional code to record state changes, fire events, and so on can soon make a difference.</p>  <p>Consider this simple entity type:</p>  

``` c#
public class Foo
{
    public int Id { get; set; }
    public int Prop1 { get; set; }
    public int Prop2 { get; set; }
    public string Prop3 { get; set; }
    public string Prop4 { get; set; }
}
```

<p>And consider performing the following update to each of 10,000 tracked entities and then calling SaveChanges:</p>

``` c#
var prop1 = foo.Prop1;
foo.Prop1 = foo.Prop2;
foo.Prop2 = prop1;
var prop3 = foo.Prop3;
foo.Prop3 = foo.Prop4;
foo.Prop4 = prop3;
```

<p>On my (rather old) laptop this took approximately 6.3 seconds. Adding virtual to every property to enable change-tracking proxies and repeating resulted in a time of approximately 6.2 seconds. So not a huge performance hit for using proxies, but really not a very significant gain either.</p>  <p>Now consider the case where an app makes multiple changes to the same property before calling SaveChanges—perhaps because a value is being re-calculated or a counter is being incremented as application state changes. If the update above is performed three times on each entity before SaveChanges is called it still takes 6.3 seconds when using snapshot change-tracking, but now takes 6.6 seconds when using change-tracking proxies. Still not much of a difference, but the change-tracking proxies are now starting to perform worse than the snapshot change-tracking.</p>  <p>Finally, what happens if the app sets properties to the values that they already have, such as might happen when copying values from DTOs, forms, or similar? Consider performing this update on each of 10,000 tracked entities and then calling SaveChanges:</p>  

``` c#
foo.Prop1 = foo.Prop1;
```

<p>With snapshot change-tracking the whole process takes about 0.1 seconds on my laptop. This makes sense because nothing is changing and so SaveChanges writes nothing to the database. But with change-tracking proxies the same thing takes about 5.2 seconds! Why? Because change-tracking proxies mark a property as modified whenever any value is written to it. You could make a very strong case that this is broken behavior and I wouldn't argue against you. The reason it is this way is that EntityObject entities behave this way, the IPOCO interfaces where then extracted from EntityObject, and change-tracking proxies were implemented using IPOCO.</p>  <h3>This seems very biased…</h3>  <p>Yes, it is. The performance numbers in particular are very biased. It would be very easy to come up with many scenarios where change-tracking proxies perform better than snapshot change-tracking. Indeed, the EF team have performance micro-benchmarks that run every night and demonstrate this. But the point still remains—change-tracking proxies are not a general solution for better performance, and they shouldn't be used as such.</p>  <h3>Summary</h3>  <p>In summary, my advice is simple: use POCO entities optionally with virtual navigation properties for lazy loading. Only consider using change-tracking proxies if you understand their issues and you have a really compelling reason to use them. This usually means only use change-tracking proxies have you have profiled your app and found snapshot change tracking to really be a bottleneck.</p>  
