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


--------

package com.tillster.exp;

import java.util.logging.Level;
import java.util.logging.Logger;

public class BadPractices {
    private BadPractices() { }
    private static final Logger log = Logger.getAnonymousLogger("BadPractices");

    public ProductOfferingResult getProductOffering(String tenant, String storeName) {
        /*
          TODO: We need to go back and rewrite this method. I'm not even quite sure how it will behave. The three biggest
                poor practices are: First, it calls a method that may return an exception instead of throwing it. That
                forces us to test the returned value for an exception, then throwing it. This is a very bad practice.
                The second problem is casting it as Throwable rather than Exception. So I'm not even sure if the
                catch(Exception) clause will catch it. I can explain why only after mentioning the third poor practice,
                which is putting a return statement in a finally block. If an uncaught exception reaches this block, it
                can't possibly return a value without swallowing the Exception. I'm pretty sure it doesn't swallow the
                Exception, but when I tried moving the return statement out of the finally block, I got a compiler error
                on the "throw (Throwable) latestMenuResult;" statement that said "Unhandled Exception." Changing
                Throwable to Exception makes the compiler error go away, which implies that this exception wasn't
                getting caught by the catch(Exception) statement, but will now get caught by it. Which means the method
                was propagating an undeclared checked exception. I'm surprised the compiler even allows this, since it
                seems to be bypassing the rules of propagating checked Exceptions. And I'm not sure if it was an
                intentional effort to avoid the catch block, or just a bug. In any case, when we fix this, we should
                start by moving the return statement out of the finally block. I won't do it now, because that would
                require testing that's outside the scope of the issue I'm working on, and any changes are at risk of
                unintentionally changing its behavior.
        */

        log.log(Level.FINE, "Entering: MobileM8WebService.getProductOffering()");

        ProductOfferingResult productOfferingResult = new ProductOfferingResult();
//        productOfferingResult.setStoreName(storeName);

        try {

            IStoreManager storeManager = getStoreManager();
            Object latestMenuResult = storeManager.getLatestMenu();

            IProductOffering productOffering = null;

            if (latestMenuResult instanceof Exception) {
                throw (Throwable) latestMenuResult;
            } else if (latestMenuResult instanceof IProductOffering) {
                productOffering = (IProductOffering) latestMenuResult;
            }

            if (productOffering != null) {
                process(productOffering);
            } else {
                productOfferingResult.setStatusCode(400);
            }

//        } catch (StoreConfigurationException sce) {
//
//            log.log(Level.WARNING, "Encountered Exception:", sce);
//            productOfferingResult.setStatusCode(404);

        } catch (RuntimeException e) {
            log.log(Level.SEVERE, e.getLocalizedMessage());
            productOfferingResult.setStatusCode(500);// , e);

        } finally {
            log.log(Level.FINE, "Leaving: MobileM8WebService.getProductOffering()");
            return productOfferingResult;
        }
    }

    private void process(IProductOffering offering) { }

    private class ProductOfferingResult {
        private void setStatusCode(int code) { }
    }

    private StoreManager getStoreManager() throws StoreConfigurationException {
        return new StoreManager();
    }

    private interface IStoreManager {
        Object getLatestMenu();
    }

    private class StoreManager implements IStoreManager {
        @Override
        public Object getLatestMenu() {
            if (externalFactor()) {
                return new Exception("Very Bad Practice!");
            }
            return new IProductOffering() {};
        }
    }

    private boolean externalFactor() { return true; }

    private interface IProductOffering {}

    private class StoreConfigurationException extends Exception { }
}
 
