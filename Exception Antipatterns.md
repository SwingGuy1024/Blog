# Exception Anti-Patterns #

These are examples of poor handling of exceptions, all taken from actual production code. Exceptions are often an overlooked issue. Developers will go into a task with a clear idea of how to solve a particular problem, but won't have a clear idea of how to respond if something goes wrong. So, on a multi-person team, often each developer will follow a different approach, so a project will follow many different philosophies for how to handle exceptions.

If you're not clear on how to handle exceptional cases, there's actually a simple rule to follow. If a function received valid input, it should return a valid result. If it receives invalid input, it should throw an Exception. *The proper behavior for invalid input is to throw an exception.*

## 1 Catch, Log, Return null ##

    public HtmlEmailBuilder getEmailBuilder( /* several parameters */ ) {
        HtmlEmailBuilder builder = new HtmlEmailBuilder();
        try {
            // ~150 lines of code deleted.

                return builder;
            }
        } catch (Exception e) {
            LOGGER.error("getEmailBuilder", e);
        }
        return null;
    }

I was asked to debug a NullPointerException in a web service, but the stack trace in the bug report didn't help, because it was caused by the null value returned by this method. Of course, a more useful stack trace was right there in the log, but whoever filed the bug didn't know that, so I got the wrong stack trace in the bug report.

I'm not sure what the developer was thinking in returning null, which is usually a useless value. Maybe s/he assumed that the calling method would test it for null. But there's really no point to this, especially in a web server. If the developer had just ignored the exception, it would have been logged anyway, and my bug report would have had the correct stack trace.

A second problem is that it catches Exception, rather than a more specific Exceptoin. When I see this, I have no way of knowing if a checked exception is getting thrown, so I always change it to RuntimeException, to see if I get a compiler error. In this case, I didn't.

Was the developer was just being conscientious, thinking that it's a better practice to catch and log any Exceptions? But it's not. The NullPointerException got logged. That happens automatically. As developers, we should trust that the calling code will correctly log any exceptions that our code throws. And if they don't, that's the first bug that needs to get fixed.

So when is null an acceptable return value? Looking through the APIs that come with the JDK, you may notice that null is typically returned when a method is searching for something that might not be there, like the `Map.get()` method. Since JDK 1.8, many new methods will now return an Optional instead. But Optional is not a good alternative for the method above. It shouldn't be used to mean something went wrong. The method above is supposed to always return a valid object. It will only return null if there's a bug in the code, or if it recieved invalid input. The proper behavior for invalid input is to throw an Exception, not to return null.

## No Exceptions in Released Code ##

I've worked on projects where we had a rule that exceptions were forbidden in released code. This led to a lot of code where we checked nearly all returned values for null. This led to some comical warnings issued by my IDE. For example I often saw cases like this:

    public void someMethod(Thing someThing, ...) {
        Widget widget = someThing.getWidget();
        ...
        if (someThing != null) {
            someThing.someMethod(...);
        }

My IDE would warn me that `someThing != null` is always true. It knows because if it was null, the previous call to `getWidget()` would have thrown a NullPointerException. I saw this warning all over the place.

Why did they do this? While this particular application was a client-server application, it wasn't a browser client. It was a stand-alone client written in Java/Swing. So the thinking was that an exception in the client would be seen only by the client, and wouldn't go into the server logs anyway. So the customer would see the exception but we wouldn't. But it would have been a simple task to report all client exceptions back to the server, where we could debug them.
