# Exception Anti-Patterns #
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

I was asked to debug a NullPointerException in a service, with a stack trace. But that exception didn't help, because it was caused by 
the null value returned by this method. Of course, a more useful stack trace was right there in the log, but whoever filed the
bug had no idea that the preceeding exception was the important one to log, so I got the wrong stack trace in the bug report.

I'm not sure what the developer was thinking in returning null, which is a useless value. Maybe s/he assumed that the calling method would test it for null. But there's really no point to this in a service. If the developer had just ignored the exception, it would have been logged anyway, and my bug report would have had the correct stack trace.

A second problem is that it catches Exception. When I see this, I have no way of knowing if there's a checked exception that's getting thrown, so I always change it to RuntimeException, to see if I get a compiler error. In this case, I didn't.

Maybe the developer was just being conscientious, and thought that it's a better practice to catch and log any Exceptions. But it's not. Exceptions will get logged. Unless we can recover form an exception, we should trust that the calling code will correctly log any exceptions that our code throws. And if they don't, that's the first bug that needs to get fixed.

