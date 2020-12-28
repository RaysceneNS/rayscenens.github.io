---
title: "One simple trick for deleting entities using Entity Framework"
tags: [Entity Framework]
---

I want to show you a simple extension method that can be used to make your Entity Framework code more readable.

## Mark Entities for Removal without Prior Retrieval

Usually in order to remove an item from a DBSet in entity framework you need to first query for the entity and then subsequently call delete on the entity. Sometimes all you really want to do is delete the item without incurring the overhead of a query. In this case you can use the .Attach() method to bypass the entity retrieval.

```c#
Customer cust = new Customer();
cust.CustId = "A123";
db.Customers.Attach(cust);
db.Customers.Remove(cust);
db.SaveChanges();
```

## EF RemoveWhere() extension

This is a small but handy extension method for those times when you are working with Entity Framework and need to remove all items from the DBSet that match a predicate.

This is the extension method:

```c#
public static class DbSetExtensions
{
    /// <summary>
    /// Query the set for matches. All matching items are marked for deletion.
    /// </summary>
    /// <typeparam name="TType"></typeparam>
    /// <param name="set"></param>
    /// <param name="match"></param>
    public static void RemoveWhere<TType>(this IDbSet<TType> set, Expression<Func<TType, bool>> match) where TType : class
    {
        var results = set.Where(match);
        foreach (var result in results)
        {
            set.Remove(result);
        }
    }
}
```

That allows you to write code like:

```c#
items.RemoveWhere(w=>w.FirstName=="John");
```

Which will mark all entities that match our predicate as 'deleted'. We then call ```SaveChanges()``` to materialize the deletions in the datastore. One drawback to this method is that we will incur the cost of the where query before the deletion and depending on your transaction level this may introduce deadlocks.
