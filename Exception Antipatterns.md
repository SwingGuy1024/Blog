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

I was asked to debug a NullPointerException, with a stack trace. But that exception didn't help, because it was caused by 
the null value returned by this method. Of course, a more useful stack trace was right there in the log, but whoever filed the
bug had no idea that the preceeding exception was the important one to log, so I got 
