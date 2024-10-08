# Examples of Bad Usage of the Optional Class

Many people, including me, were excited that an Optional class was introduced in Java 8. As soon as I saw it, I was disappointed by how verbose it was, and puzzled by the strange and long-winded approach. At the time, I misunderstood its purpose, as did a lot of other people. Many people began to use it extensively. It's now very widespread, and it has led to some amazingly bad code, written by people who don't understand what it's for. In certain limited situations it's actually very useful, although I rarely actually use myself. I'm planning to expand this to a more comprehensive take on Java's Optional class, but for now, I offer some examples of very bad usages of the Optional class. (Some of this is taken from a [StackOverflow question](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type) that I answered. This expands on my answer.)

## Optional's Purpose
A [blog post by Brian Goetz](http://mail.openjdk.java.net/pipermail/lambda-dev/2012-September/005952.html) does a good job of illustrating why Optional was invented. It was introduced at the same time as functional programming, and it's helpful because the return expression in example B is cleaner than the three lines of code in example A:


***Example A: Clumsy functional code***

    Method matching =
      Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
      .stream()
      .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
      .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
      .filter(m -> Objects.equals(m.getReturnType(), returnType))
      .getFirst();                        // getFirst() does not return Optional. May return null.
    
    if (matching == null)
      throw new InternalError("Enclosing method not found");
    return matching;

***Example B: Cleaner code, in a single long expression***

    return Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
      .stream()
      .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
      .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
      .filter(m -> Objects.equals(m.getReturnType(), returnType))
      .findFirst()                        // findFirst() returns Optional<Method>.
      .getOrThrow(() -> new InternalError("Enclosing method not found"));

Brian Goetz shows how Optional is needed to support function chaining. It allows a chain of calls to continue even if a method returns null. This is used widely in functional programming, which was added to Java at the same time. That's the only place where I use it. It's used here in the `findFirst()` method, which, after all the filtering, might not even have a value. The `findFirst()` call, which returns `Optional<T>` replaced the `getFirst()` method from example A, which may return null. The Optional in turn lets us continue the chain of methods. Consequently, it makes sense to use Optional as a method return type, but only when null is an expected possible return value.

Here's another example, where optional is used as the return value of a function that is passed to a method. It shows up in the very useful `Optional.flatMap()` method, but not in the `Optional.map()` method. Here are their signatures:

    public <U> Optional<U> flatMap(Function<? super T, ? extends Optional<? extends U>> mapper)
    public <U> Optional<U> map(Function<? super T, ? extends U> mapper)

The two methods do the same thing, but `flatMap()` takes a Function that returns Optional, while `map()` takes one that doesn't. The two methods do the same thing, but the user has a choice. This is a useful usage pattern. It lets us limit the use of Optional to methods that might actually return null. 

But all too often, it gets used where `Required` might be a better name. It's also used a lot of places where it really isn't very helpful, or it's so verbose that it makes the code clumsy. Sometimes it's used with the intention of preventing a NullPointerException, but it's used so poorly that it actually creates new ways to introduce bugs. Rather than add to the endless debate about how to use it, I'd like to illustrate just how badly it's getting used. Here are some examples, all taken from actual production code.

## Bad Code Examples

### Example 1: Needless Verbosity

    01 private void someMethod(Widget widget, ... <other parameters>) {
    02   Optional<Widget> widgetOpt = Optional.of(widget);
    03   if (!widgetOpt.isPresent()) {
    04     throw new BusinessException("Missing Widget");
    05   }
    06   // ... widgetOpt is never used again, even though it could be.
    07   // ... several lines later
    08   Optional<Widget> widgetMaybe = Optional.of(widget); // this duplicates the previous Optional!
    09   processData(widgetMaybe);
    10   // ...more code
    11 }

What's going on here? On lines 2 and 3, the coder is taking two lines to do a simple null check. It wraps `widget` in an Optional, then throws it away and wraps the same value inside a second Optional on line 8. 

The first use is needlessly verbose, and seems to be done just to avoid ever using the `null` keyword in the code, as if it's use is now taboo. Lines 02 - 05 could have been written like this:

    02   if (widget == null) {
    03     throw new BusinessException("Missing Widget");
    04   }

This is cleaner, more readable, and even faster. 

The second use of Optional isn't used for function chaining, but merely because someone wrote a method that takes an `Optional<Widget>` as a parameter.

To make matters worse, there's a repeated bug in lines 02 and 08 that didn't get caught. It says `Optional.of(widget)`, but it should say `Optional.ofNullable(widget)`! So if a null value ever gets passed into this method, it will throw a `NullPointerException` on line 2 instead of the `BusinessException` thrown on line 04. But nobody caught this bug because the code that called this *private* method had already made sure that widget was not null, so the test wasn't even necessary.

### Example 2: Misleading Return values

Here's a class that was generated by Swagger 2.0, using their Spring Server generator, with the Java 1.8 option turned on:

    @Controller
    public class MenuItemApiController implements MenuItemApi {
        private final ObjectMapper objectMapper;
        private final HttpServletRequest request;

        @Autowired
        public MenuItemApiController(ObjectMapper objectMapper, HttpServletRequest request) {
            this.objectMapper = objectMapper;
            this.request = request;
        }

        @Override
        public Optional<ObjectMapper> getObjectMapper() {
            return Optional.ofNullable(objectMapper);
        }

        @Override
        public Optional<HttpServletRequest> getRequest() {
            return Optional.ofNullable(request);
        }
    }
This has two private final members, and two getters that wrap the return values in an Optional. But take a look at that constructor. It's annotated with @Autowired. Spring will automatically instantiate valid values for both of the parameters and inject them into the constructor. And they're final, so they'll never change to null. The methods are returning objects that can never be null, wrapped inside an Optional. Does the use of Optional here add clarity to the API? It actually does the opposite: It suggests that two never-null objects might actually be null, encouraging users to write code to check for null values that they'll never see.

This also breaks the convention I discussed in the opening, where the user is given the choice of two methods, one of which returns an Optional. In that convention, the user can limit the use of Optional to the cases where it might actually see a null value. In this example, the code makes that choice for the user, and makes the wrong one. 

### Example 3: Misleading Parameters

Many people have written guidelines recommending against using Optional as parameter methods, but people do it anyway, saying that it forces the caller to think about whether the object is null. But I've never seen a case where it actually helps. Here's one way this goes wrong. (There are others, below.)

    1 private void someMethod(Optional<Widget> widgetOpt) {
    2   if (!widgetOpt.isPresent()) {
    3     throw new BusinessException("Missing Widget");
    4   }
    5   // ... (More code)

This method uses `Optional` to mean `Required`. So why is the Widget wrapped in an Optional? And what clarity is being added to the API by using Optional? To anyone reading the JavaDocs, the API implies that null is a valid input value. But it's clearly not. This method has an interface that's both misleading and more verbose to use, since the user must now write `someMethod(Optional.ofNullable(widget))` instead of `someMethod(widget)` A good method signature should take care of boilerplate details needed to call the method, to make the call as simple as possible. By using an Optional parameter, this does the opposite. It also introduces a new kind of bug that wouldn't even be possible without the Optional parameter: The calling method could incorrectly call it by saying `someMethod(Optional.of(widget))` which will throw a `NoSuchElementException` if widget is null. 

Prior to the invention of Optional, the author could have made it clear that the parameter is required by giving it a more descriptive name:

    1 private void someMethod(Widget requiredWidget) {
    2   if (requiredWidget == null) {
    3     throw new BusinessException("Missing Widget");
    4   }
    5   // ... (More code)
This is cleaner and easier to call. The addition of Optional to the interface doesn't do much beyond giving us an alternate way to test for null, but which puts greater demands on the user.

### Example 4: Seemingly Sensible, but Pointless

Here's a case where the use of Optional clearly does not mislead the user.

    1 private void someMethod(Optional<Widget> widgetOpt) {
    2   final Widget widget = widgetOpt.orElseGet(() -> getDefaultWidget());
    3   // ... (More code)

Here, finally, we can say that Optional is adding clarity to the API. The Widget parameter is actually allowed to be null. Here, the API does not mislead anyone. Of course, users must be careful not to call `Optional.of(widget)`. Instead, they should use `Optional.ofNullable(widget)` or `Optional.empty()`, but that's a fail-fast mistake, so it will get caught early. Unfortunately, so many developers wrap required parameters inside Optional, that the meaning of this occasional valid use will often get lost anyway.

However, does Optional really add any value here? If the authors are using Optional to prevent a NullPointerException, this is completely unnecessary. The method body already prevents that. If we remove the Optional wrapper, the method looks like this:

    1 private void someMethod(final Widget widgetOrNull) {
    2   final Widget widget = (widgetOrNull == null) ? getDefaultWidget() : widgetOrNull;
    3   // ... (More code)

Either way, the parameter is null-safe, but the second way is both easier to call, and less error-prone—the user can't mistakenly call `Optional.of()`. Does the use of Optional force the caller to think about how to call it? Who cares? This method can't be called incorrectly! You dont need Optional to make this null-safe.

Simply renaming the parameter provides the same information as Optional. Before Optional was invented, not many people did this, which is probably a shame, because it adds clarity without adding verbosity. Look at the two ways to call a method like this:

    someMethod(Optional.ofNullable(widget));  // Verbose, each time it gets called.
    someMethod(Optional.of(widget));          // Easy to make this mistake.
or

    someMethod(widget);                       // Simpler and can't be called incorrectly.

So once again, Optional creates a new way to introduce a bug.

And, before Optional was invented, there was already a widely-used way to specify a parameter as optional: Overloading!

    private void someMethod() { someMethod(getDefaultWidget()); }
    private void someMethod(Widget requiredWidget) { ... }

Of course, it's still possible for someone to call the second method with null. If this is a concern, you can still prevent it by writing your code this way:

    private void someMethod() { someMethod(null); }
    private void someMethod(final Widget widgetOrNull) {
      final Widget widget = (widgetOrNull == null) ? getDefaultWidget() : widgetOrNull;
      // ... (More code)
    }

Some people may find this strange. Maybe their right. But it works.

In short, Optional is a bad and ineffective alternative to thoughtfully written code.

### Example 5: Pointless
    1 private void someMethod(Optional<Widget> widgetOpt) {
    2   if (!widgetOpt.isPresent()) {
    3     throw new NullPointerException();
    4   }
    5   Widget widget = widgetOpt.get();
    6   widget.doSomething();
    7 }
    
Yeah. I've seen this. Once again, Optional means Required. And lines 2-4 are unnecessary. Before `Optional` was invented, this would have behaved exactly the same way:

    1 private void someMethod(Widget requiredWidget) {
    2   requiredWidget.doSomething();  // may throw NullPointerException
    3 }
    
This one also throws a NullPointerException if the parameter is null. But it takes one line to do what five lines did in the first case. It also gave us an opportunity to clarify the required nature of the parameter with a better name. Although "required" should be implicit.

By the way, in spite of the possible NullPointerException, this is still a good way to write the method. It's fail fast, meaning that if a null value is passed, the code will immediately throw an exception, probably in the development phase where you can catch it and fix it before it reaches production.

### Example 6: Also Pointless

    public WidgetRequest getWidgetRequestByToken(String token) {
        return widgetReqTblDao.findByNamedParam("token", token).orElse(null);
    }
    

In the class where I found this code, the `findByNamedParam()` method, which returns an Optional, is only used in two places, both in the same class, and is used the same way in each case. It returns an Optional, which this method converts to a null if it's empty. Of course, it would have been simpler to just return null instead of Optional. Farther up the call stack, the object is checked for null. It would also have made sense to send the Optional all the way back. But as written, the use of Optional here is harmless but pointless.


### Example 7: Self Defeating

This one, I don't even know how it made it into production. In this example, a NullPointerException is thrown on the second line. There are four places in the line where a null will cause an Exception. Can you figure out where it comes? (The getFooOpt() method returns `Optional<Foo>`.) 

    thing.setSomeProperty(result, widget.getSomeProperty());
    thing.setFooForResult(result, widget.getFooOpt().get()); // NullPointerException

If you can't narrow it down, notice that `thing` and `widget` can't be null, or the exception would have been thrown on the previous line. My IDE issues a warning for the call to `get()`, saying *'Optional.get()' without 'isPresent()' check*. But that's not the problem, because an empty `Optional.get()` will throw a NoSuchElementException. So it's clear that the problem is that the `Optional<Foo>` returned by `getFooOpt()` is itself null.

Here's the class:

    public class Widget {
        // ... (other properties, etc)
        private Optional<Foo> foo;

        public Optional<Foo> getFooOpt() {
            return foo;
        }

        public void setFoo(Optional<Foo> foo) {
            this.foo = foo;
        }
    }

Of course it's null! They never initialize the Optional member. When an Object is an Optional instance, it's just as likely to be null as any other object. So if the developer was using Optional to avoid a `NullPointerException`, it didn't work. (Of course, Optional wasn't written to solve this problem, and as this example illustrates, it doesn't.) Ironically, the Optional wrapper is never optional.

If your member object is Optional, it needs to be initialized:

    private Optional<Foo> foo = Optional.empty();

But what does your setter look like? It's still possible to set the value to null, so do you guard against that?

    public void setFoo(Optional<Foo> foo) {
        this.foo = (foo == null)? Optional.empty() : foo; // clumsy! 
    }

Now you're worrying about two null values, the Foo that may be null and the Optional that holds it. Setters should be very simple to write. 

But two simple changes return us to simplicty, and buy us robustness. First, don't store an Optional. Second, don't take one as a parameter.

    public class Widget {
        // ...
        private Foo foo;                     // No longer wrapped in Optional, might be null

        public void setFoo(Foo fooOrNull) {
            foo = fooOrNull;                 // No need to check for null, null value is ok
        }
        
        public Optional<Foo> getFooOpt() {
            return Optional.ofNullable(foo); // The only place where we use Optional
        }
    }

Here, the code and the method signatures are as simple as they can get. We never need to check for null. Also, simplicity buys us robustness. Optional is only used in the one place where it's actually needed, and it's used in a way that guarantees it can't be null. This is a good illustration of the KISS principle -- Keep it Simple, Stupid!

The final irony in this example is this: In order to implement this in a way that guarantees it won't produce a NullPointerException, we actually had to remove Optional from the setter. So this is still another new way for Optional to cause an exception.

(By the way, the possibility of the getter returning null should have been caught by unit tests. Your unit tests should always test for proper behavior when given bad input. Proper behavior for bad input usually means throwing an exception, so these tests are easy to write. But they often get overlooked.)

If you  insist on keeping an Optional member in the class, there's still a reliable way to do this. Your class can't implement serializable, but it will work. You still need to keep Optional out of your parameters.


    public class Widget {
        // ...
        private Optional<Foo> foo = Optional.empty();

        public void setFoo(Foo fooOrNull) {
            foo = Optional.ofNullable(fooOrNull);
        }
        
        public Optional<Foo> getFooOpt() {
            return foo;
        }
    }
    
What these two implementations have in common is that they take total control over where and how Optional is instantiated. The user can't make the mistake of calling `Optional.of()` instead of `Optional.ofNullable()`.

Some people will object, saying an Optional parameter will force the user to think about whether the parameter is null or not. But why force the user? This way, the user doesn't have to worry about it. Either way, it works.

### Quick Takes:
**1. This code is fine, but it doesn't take advantage of what Optional has to offer.**

       private void deleteLegacyUserIfExists(String email) {
           LegacyUser legacyUser = legacyUserService.getLegacyUser(email).orElse(null);
           if (legacyUser != null) {
               legacyUserService.deleteLegacyUser(email);
           }
       }
   
It works, but you may change it to this:

    private void deleteLegacyUserIfExists(String email) {
        legacyUserService
            .getLegacyUser(email)
            .ifPresent(() -> legacyUserService.deleteLegacyUser(email));
    }

**2. This one declares an Optional return, but may still return null.**

    public static Optional<POSType> getPOSType(String posName) {
        if (StringUtils.isNotBlank(posName)) {
            return Arrays.stream(POSType.values())
                    .filter(type -> type.toString().equalsIgnoreCase(posName))
                    .findFirst();
        }
        return null;
    }
    
If the `findFirst()` method doesn't find anything, it correctly returns an empty `Optional.` But if the input parameter is missing, there's nothing to find, so it should return `Optional.empty().` But it returns `null` instead! I had to fix the ensuing `NullPointerException` that Optional, once again, had failed to prevent.

**3. This one took way too much code to do something very simple.**

Here's a simple requirement: Determine if a user's ticket (a String) is regular or admin, based on its prefix. Every ticket starts with one of these two prefixes. I would have written the code like this:

    public static boolean isLocalAdmin(String ticket) {
        return StringUtils.startsWith(ticket, ADMIN_PREFIX); // ADMIN_PREFIX is a String constant.
    }
    
    public static boolean isUser(String ticket) {
        return StringUtils.startsWith(ticket, TICKET_PREFIX); // TICKET_PREFIX is a String constant.
    }

But somebody got the idea that this task was a bit bigger. While I try to discourage overuse of Optional, I encourage the use of `enum,` but the `enum` in the code below is pointless. Its two values were used nowhere else in the project; nor was its big public static method, `getTicketType().` Even without Optional, this code would have been clumsy and verbose, but Optional only makes it worse. None of this extra code does anything more than my two simple methods above, which use the same API.

    // This enum is an inner class of a larger class, which defines the two  "x_PREFIX" String
    // constants used here. Their values aren't important. 
    public enum TicketType { // This public enum could have been private. It's not used anywhere else.
        LOCAL_ADMIN,
        USER;

        // This could have been private, as it's only called by the two public static methods below:
        public static Optional<TicketType> getTicketType(String ticket) {
            Optional<TicketType> ticketTypeOptional = Optional.empty();
            if (StringUtils.startsWith(ticket, ADMIN_PREFIX)) {
                ticketTypeOptional = Optional.of(LOCAL_ADMIN);
            } else if (StringUtils.startsWith(ticket, TICKET_PREFIX)) {
                ticketTypeOptional = Optional.of(USER);
            }
            return ticketTypeOptional;
        }
    
        public static boolean isLocalAdmin(String ticket) {
            Optional<TicketType> ticketTypeOptional = getTicketType(ticket);
            return ticketTypeOptional.isPresent() && ticketTypeOptional.get() == LOCAL_ADMIN;
        }
    
        public static boolean isUser(String ticket) {
            Optional<TicketType> ticketTypeOptional = getTicketType(ticket);
            return ticketTypeOptional.isPresent() && ticketTypeOptional.get() == USER;
        }
    }
