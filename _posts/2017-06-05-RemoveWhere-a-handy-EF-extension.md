---
layout: post
title: RemoveWhere() a handy EF extension.
tags: [Entity Framework]
---

# Writing RemoveWhere in EF6

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