---
title: How-to seed data in an Entity Framework Core 2.0 project
excerpt: "The data seeding mechanism in Entity Framework Core has changed from the implementation in EF6. This is a quick-start method to implement data seeding in your projects."
tags: [.Net]

---

# Seeding data in Entity Framework Core

Recently I was playing around with Entity Framework Core. And needed to seed some data, but found that the seeding support has changed quite a bit from how it was setup in Entity Framework 6.

We need to call SeedData.Initialize passing in a data context, like so.

```c#
    public static void Main(string[] args)
    {
        var host = BuildWebHost(args);

        using (var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;

            var context = services.GetRequiredService<AppDbContext>();
            context.Database.EnsureCreated();
            SeedData.Initialize(context);
        }
        host.Run();
    }
```

It is up to us to perform the seeding which I've done with a simple class that contains the various Seed data functions.

```c#
    public static void Initialize(AppDbContext context)
    {
        if (context.Items.Any())
                return;

        context.Items.Add(new Item{ Name="Foo", Title="Stuff" ...etc... });
        context.SaveChanges();
    }
```

At this point our items are added to the data context and calling SaveChanges() commits them.