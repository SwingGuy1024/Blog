# Examples of Bad Usage of the Optional Class

Many people, including me, were excited that an Optional class was introduced in Java 8. As soon as I saw it, I was disappointed by how verbose it was, and puzzled by the strange and long-winded approach. At the time, I misunderstood its purpose, as did a lot of other people. Many people began to use it extensively. It's now very widespread, and it has led to some amazingly bad code, written by people who don't understand what it's for. In certain limited situations it's actually very useful, although I rarely actually use myself. I'm planning to expand this to a more comprehnsive take on Java's Optional class, but for now, I offer some examples of very bad usages of the Optional class. (Some of this is taken from a [StackOverflow question](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type) that I answered. This expands on my answer.)

## Optional's Purpose
Optional was introduced at the same time as functional programming. It's helpful because the return expression in example B is cleaner than the three lines of code in example A:


***Example A: Clumsy functional code***

    Method matching =
      Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
      .stream()
      .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
      .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
      .filter(m -> Objects.equals(m.getReturnType(), returnType))
      .getFirst();
    if (matching == null)
      throw new InternalError("Enclosing method not found");
    return matching;

***Example B: Cleaner code, in a single long expression***

    return Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
      .stream()
      .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
      .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
      .filter(m -> Objects.equals(m.getReturnType(), returnType))
      .findFirst()
      .getOrThrow(() -> new InternalError("Enclosing method not found"));

(This example comes courtesy of a [blog post by Brian Goetz](http://mail.openjdk.java.net/pipermail/lambda-dev/2012-September/005952.html)) My point is that Optional is needed to support functional programming, which was added to Java at the same time. That's the only place where I use it. It's used here in the `findFirst()` method, which, after all the filtering, might not even have a value. The `getFirst()` method from example A, which may return null, gets replaced by the `findFirst()` method, which wraps the null value inside an Optional. The Optional, in turn, lets us continue the chain of methods.

Another place where it gets used is in the very useful `Optional.flatMap()` method, but not in the `Optional.map()` method. Here are their signatures:

    public <U> Optional<U> flatMap(Function<? super T, ? extends Optional<? extends U>> mapper)
    public <U> Optional<U> map(Function<? super T, ? extends U> mapper)

The two methods do the same thing, but `flatMap()` takes a Function that returns Optional, while `map()` takes one that doesn't. This is a useful way to write an API. The two methods do the same thing, but the user has a choice. So Optional will only be used when a method might actually return null.

In both of these examples, the API offers two methods, one of which uses Optional. This is a useful usage pattern.

But all-too-often it gets used where `Required` might be more descriptive. It's also used a lot of places where it really isn't very helpful, or it's so verbose that it's clumsy. Rather than add to the endless debate about how to use it, I'd like to illustrate just how badly it's getting used. Here are some examples, all taken from actual production code.

## Bad Code Examples

### Example 1: Needless Verbosity

    01 private void someMethod(Widget widget, ... <other parameters>) {
    02   Optional<Widget> widgetOpt = Optional.of(widget);
    03   if (!widgetOpt.isPresent) {
    04     throw new BusinessException("Missing Widget");
    05   }
    06   // ... widgetOpt is never used again, even though it could be.
    07   // ... several lines later
    08   Optional<Widget> widgetMaybe = Optional.of(widget); // this duplicates the previous Optional!
    09   processData(widgetMaybe);
    10   // ...more code
    11 }

What's going on here? One lines 2 and 3, the coder is taking two lines to do a simple null check. Then it throws `widgetOpt` away, and it creates a second Optional of the same value on line 8. 

The first use is needlessly verbose, and seems to be done just to avoid ever using the `null` keyword in the code, as if it's use is now taboo. Lines 02 - 05 could have been written like this:

    02   if (widget == null) {
    03     throw new BusinessException("Missing Widget");
    04   }

This is cleaner, more readable, and even faster. 

But the second use of Optional is necessary. But it's necessary not because they're doing functional programming, but merely because someone wrote a method that takes an `Optional<Widget>` as a parameter.

To make matters worse, there's a repeated bug in lines 02 and 08 that didn't get caught. It says `Optional.of(widget)`, but it should say `Optional.ofNullable(widget)`! So if a null value ever gets passed into this method, it will throw a `NullPointerException` instead of the `BusinessException` thrown on line 04. But nobody caught this error because the code that called this *private* method had already made sure that widget was not null, so the test was unnecessary.

### Example 2: Misleading Return values

Here's a class that was generated by Swagger 2.0, using their Swing server generator, with the Java 1.8 option turned on:

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
This has two private final members, and two getters that wrap the values in an Optional. But take a look at that constructor. It's annotated with @Autowired. Spring will automatically instantiate valid values for both of the parameters and inject them into the constructor. And they're final, so they'll never change to null. The methods are returning objects that can never be null, wrapped inside an Optional. Does the use of Optional here add clarity to the API? It actually does the opposite: It suggests that two never-null objects might actually be null, encouraging users to write code to check for null values that they'll never see.

This also breaks the convention I discussed in the opening, where the user is given the choice of two methods, one of which returns an Optional. In that convention, the user can limit the use of Optional to the cases where it might actually see a null value. In this example, the code makes that choice for the user, and makes the wrong one. 

### Example 3: Misleading Parameters

Many people have written guidelines recommending against using Optional as parameter methods, but people do it anyway. A common variation of the code in example 1 looks like this:

    01 private void someMethod(Optional<Widget> widgetOpt) {
    02   if (!widgetOpt.isPresent) {
    03     throw new BusinessException("Missing Widget");
    04   }
    05   // ... (More code)

Again I have to ask: What clarity is being added to the API by using Optional? To anyone reading the JavaDocs, the API implies that null is a valid input value. But once you look at the code, it's clearly not. A case could be made for using Optional when null is actually allowed, but so many methods are written like this one that the point will get lost. (Not that it's a good idea anyway.) But this case, where an Optional is required where a null value throws an exception, results in a misleading API that's more verbose to call, since the user must now write `someMethod(Optional.ofNullable(widget));` instead of `someMethod(widget);` A method signature should take care of boilerplate details needed to call the method, to make the call as simple as possible. By using an Optional parameter, this does the opposite. This method uses `Optional` to mean `Required`.

### Example 4: Seemingly Sensible Use

Here's a case where it may actually make sense to use Optional on a method parameter:

    01 private void someMethod(Optional<Widget> widgetOpt) {
    02   final Widget widget = widgetOpt.orElse(getDefaultWidget());
    03   // ... (More code)

Here, finally, we can say that Optional is adding clarity to the API. Use of null instead of a Widget instance is actually allowed. Here, the API does not mislead anyone. Of course, users who choose to use null must be careful not to call `Optional.of(widget)`. Instead, they should use `Optional.ofNullable(widget)` or `Optional.empty()`, but that's a fail-fast mistake, so it will get caught early. Unfortunately, so many developers wrap required parameters inside Optional, that the meaning of this occasional valid use will often get lost anyway. And, there's a simpler way to write the API that doesn't add verbosity to the calling method:

    01 private void someMethod(final Widget widgetOrNull) {
    02   final Widget widget = widgetOrNull == null? getDefaultWidget() : widgetOrNull;
    03   // ... (More code)
    
Simply renaming the parameter will provide the same information as Optional. Before Optional was invented, not many people did this, which is probably a shame, because it adds clarity without adding verbosity. 

    someMethod(Optional.ofNullable(widget));  // The first method should be called like this.
    someMethod(widget);                       // The second may be called like this.
Furthermore, 

### Example 5: Pointless
    01 private void someMethod(Optional<Widget> widgetOpt) {
    02   if (!widgetOpt.isPresent) {
    03     throw new NullPointerException();
    04   }
    05   Widget widget = widgetOpt.get();
    06   widget.doSomething();
    07   // ... (More code)
    
Yeah. I've seen this. Once again, Optional is used for a required value. And lines 2-4 are unnecessary. It could be argued that this is more readable, because the throw is now explicit, but readability could be accomplished with a simple comment:

    01 private void someMethod(Optional<Widget> widgetOpt) {
    02   Widget widget = widgetOpt.get(); // throws NullPointerException
    03   widget.doSomething();            
    04   // ... (More code)
    
Or you could remove Optional, and get a method that's easier to call:

    01 private void someMethod(Widget widget) {
    03   widget.doSomething();  // throws NullPointerException
    04   // ... (More code)
    
All three of these methods do exactly the same thing.
