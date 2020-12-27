---
title: "Canvas App can't see ModelDrivenFormIntegration.Item at App Start"
tags: [Power Apps]
---

I started working on a canvas app using the Microsoft Power Platform. My application is placed on a Model driven form as a custom control. It was supposed to integrate with data provided by the Model Driven form and then prompt the user to either edit or create the data, presenting a separate screen for each task. At first I thought that the OnDataRefresh event would fire at the time that the model driven data finished loading in the background. It turned out that it isn’t possible to navigate to another screen from within the OnDataRefreshAction, so I needed to come up with another way.

## Solution

What I had to do to make this work wasn’t pretty, but I'm finding a lot of things in Power Apps need bizarre work arounds.

I added a Global scoped Boolean variable named ModelDrivenFormIntegrationRefreshFired. I set this at the end of the ModelDrivenFormIntegration OnDataRefresh Action, like so.

```vb
Set(ModelDrivenFormIntegrationRefreshFired, false);
Refresh([@DataSourceItems]);
Set(ModelDrivenFormIntegrationRefreshFired, true);
```

I built a start screen that I set to show first when my App starts up by setting the OnStart Action to.

```vb
Navigate('Start Screen')
```

Within the start screen I dropped on a timer control that is setup as:

- Duration = 500ms
- AutoStart = ModelDrivenFormIntegrationRefreshFired

The OnTimerEnd event is where I place the code that makes the decision about which form to present to the user based on the value of my ModelDrivenFormIntegration Item.

```vb
If(
    IsBlank(ModelDrivenFormIntegration.Item.'Primary Key'),
    Navigate('Create New Data Screen'),
    Navigate('Edit Data Screen')
)
```

So now when the application loads, it immediately displays the StartScreen, the timer on this screen is activated after the OnDataRefresh is completed. The timer then waits a small period of time, allowing Asynchronous data loading events to complete. After the OnTimerEnd event fires and the code can make a decision about which screen to show.
