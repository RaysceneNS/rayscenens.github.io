---
tags: [.Net]
---

# Create Task Asynchronous style calls from Event Handlers

If you find yourself dealing with a lot of legacy asynchronous code that uses event handlers. You may want to wrap these legacy event calls using the new .Net Task Asynchronous style wrappers.

You have a web service event wrapper that is defined in this format.

```c#
    event EventCompletedEventHandler eventCompleted;
    void callAsync(int argument, object userState);
```

Using event handler pattern the call to this web service wrapper would look like this, you provide a callback method and wire the event through the use of the += operator. Then invoke the method.

```c#
    this.callCompleted += WebService_CallCompleted;
    this.callAsync(1, node);

    private void WebService_callCompleted(object sender, AsyncCompletedEventArgs e)
    {
        //
    }
```

Let's convert this method to a form that we can use to await a Task. The first step is create a method that will return a new Task to a caller. We use a General TransferCompletion method to create an event handler that will receive the result of our async call. The handler performs basic housekeeping required to make this a Task.

1. Check for the state of the cancelled bit on the event handler, If set then the result of the call is a cancellation and we must inform the caller by setting tcs.TrySetCanceled();
1. Check for the presence of any exceptions passed to us in the Error property, if an exception is present then inform the caller by setting tcs.TrySetException(e.Error);
1. Check for the presence of any result methods to pass in the Task by calling the Function getResult(), this is the function passed to the TransferCompletion method from our caller and this will typically be () => e.Result in the case that we want to pass the result of the source event back.
1. Call the optional unregisterHandler to remove the event registration.

```c#
    public Task<ExecutionPlanningNode> CallAsync(int arg)
    {
        var tcs = new TaskCompletionSource<int>(state, TaskCreationOptions.None);
        EventCompletedEventHandler handler = null;
        handler = (sender, e) => TransferCompletion(tcs, e,
            () => e,
            () => _service.eventCompleted -= handler);
        _service.eventCompleted += handler;
        _service.callAsync(arg, tcs);
    }

    internal static void TransferCompletion<T>(
        TaskCompletionSource<T> tcs, AsyncCompletedEventArgs e,
        Func<T> getResult, Action unregisterHandler)
    {
        if (e.UserState != tcs)
            return;

        try
        {
            if (e.Cancelled)
            {
                tcs.TrySetCanceled();
            }
            else if (e.Error != null)
            {
                tcs.TrySetException(e.Error);
            }
            else
            {
                tcs.TrySetResult(getResult());
            }
        }
        finally
        {
            unregisterHandler?.Invoke();
        }
    }
```