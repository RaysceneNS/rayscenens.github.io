---
layout: post
title: How to implement the Queue Collector within an Azure function
tags: [Azure,FunctionApp]
---

## How to implement the Queue Collector within an Azure function

Here is a quick note to my future self.

Whilst working with Azure functions you may come across a situation where you need
to return multiple items to a queue, it isn't immediately obvious how to do that. Placing a message on the queue from within an
Azure Function typically involves placing the `[return: Queue("function2Queue")]` attribute on the main function body. But to add many messages
you must add a parameter decorated with `[Queue("function1Queue")] ICollector<__type__>` to the definition of the Run method.

This example creates an Http Triggered function that places multiple messages on a queue to be processed by the second function:

``` c#

    public static class Function1
    {
        [FunctionName("Function1")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = null)] HttpRequest req,
            [Queue("function1Queue")] ICollector<string> queueCollector)
        {
            queueCollector.Add("John Markson");
            queueCollector.Add("Mark Johnson");
            return new OkResult();
        }
    }

    public static class Function2
    {
        [return: Queue("function2Queue")]
        [FunctionName("Function2")]
        public static async Task Run(
            [QueueTrigger("function1Queue")] string message)
        {
            return $"C# Queue trigger function processed: {message}";
        }
    }

```
