---
title: "Implement IAuditEntity to automatically audit your entities"
tags: [Entity Framework]
---

Apply the audit information for your Entity Framework entities automatically as part of the SaveChanges() method, doing so ensures you remove the possibility that this crucial information is not managed consistently.

## Overview

Sometimes its the little things that make big differences to the quality of a design. Needing to remember to manually update the audit information within your entities whilst working with an ORM like Entity Framework will lead to inconsistent auditing of fields, this can be worse than having no audit information at all, as it can mislead your operations team. However if we design a solution that can perform this task automatically as we use the entity then we will have removed a potential source of bugs.

## Solution

Create the following interface to define the data fields that store the audit information. This will ensure a consistent layout to the fields that are audited for each entity.

```c#
/// <summary>
/// Implement this interface to denote that your entity model should automatically set the created and modified audit fields
/// </summary>
public interface IAuditEntity
{
    DateTime CreatedDate { set; }
    string CreatedBy { set; }
    DateTime ModifiedDate { set; }
    string ModifiedBy { set; }
}
```

To use this interface we override the SaveChanges() method on our DBContext so that we now call a new method AuditEntities() that is inserted into the usual process of change detection and save.

```c#
public override int SaveChanges()
{
    // Call DetectChanges to synchronize the objects with the state manager.
    this.ChangeTracker.DetectChanges();
    AuditEntities();
    return base.SaveChanges();
}
```

AuditEntities will look get any entity that has the state Added or Modified. If the entry also implements our IAuditEntry interface then we update the required audit fields.

```c#
private void AuditEntities()
{
    ObjectContext ctx = ((IObjectContextAdapter)this).ObjectContext;

    var objectStateEntryList =
        ctx.ObjectStateManager.GetObjectStateEntries(EntityState.Added | EntityState.Modified).ToList();

    string currentUser = Thread.CurrentPrincipal.Identity.Name;
    DateTime now = DateTime.UtcNow;

    foreach (ObjectStateEntry entry in objectStateEntryList)
    {
        if (entry.IsRelationship)
        {
            continue;
        }

        //only apply automatic audit information to those entities that have implemented the IAuditEntry interface.
        var auditInterface = entry.Entity as IAuditEntity;
        if(auditInterface == null)
        {
            continue;
        }

        switch (entry.State)
        {
            case EntityState.Added:
                auditInterface.CreatedDate = now;
                auditInterface.CreatedBy = currentUser;
                auditInterface.ModifiedDate = now;
                auditInterface.ModifiedBy = currentUser;
                break;

            case EntityState.Modified:
                auditInterface.ModifiedDate = now;
                auditInterface.ModifiedBy = currentUser;
                break;
        }
    }
}
```

Now whenever we modify the entity, it will automatically set the modified date to the current system date (UTC thank you) and the current user principal of the running thread. Combined with an authentication mechanism that sets the thread principal in this way we can identify the source account even if this is a web api call.