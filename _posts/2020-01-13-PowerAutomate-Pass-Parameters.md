---
tags: [Power Apps,Power Automate]
---

# Passing parameters to Power Automate from CanvasApp

Recently while developing a CanvasApp using the Microsoft Power Platform I came across an error message at the point that I was calling a Power Automate workflow. The workflow had been working up until this point but I added a new parameter, and now things weren't working.

`The input body for trigger 'manual' of type 'Request' did not match its schema definition.`

The problem here is that the parameters sent from Power Apps to Power Automate are now out of sync, to get around this you need to remove and re-add the Power Automate workflow connection from your app.

If your like me and get tired of rebinding the connections to your flows all the time then you can replace your parameters with a single JSON string in your Power Automate workflow.

Simply place a Parse JSON action at the top of your Power Automate designer and provide a schema that describes your parameters.

To pass the JSON from your App to Power Automate use the  [JSON()](https://docs.microsoft.com/en-us/powerapps/maker/canvas-apps/functions/function-json>) function to format the data parameter, like so.

```vbn
'PowerApp->Workflow'.Run(
    JSON(
        {
            foo: txtFoo.Text,
            bar: txtBar.Text
        }
    )
);
```

The other benefit to this approach is that you can also remove/refactor parameters in an existing Power Automate workflow.
