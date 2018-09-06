# Java Optional, Java's Nullness Flaw, and the Kotlin Language
Many people, including me, were excited that an Optional class was introduced in Java 8. As soon as I saw it, I was disappointed by how verbose it was, and puzzled by the strange and long-winded approach. At the time, I misunderstood its purpose, as did a lot of other people. Many people began to use it extensively. It's now very widespread, and it has led to some amazingly bad code, written by people who don't understand what it's for. In certain limited situations it's actually very useful, although I rarely actually use myself. Here's my take on Java's Optional class. (Some of this is taken from a [StackOverflow question](https://stackoverflow.com/questions/26327957/should-java-8-getters-return-optional-type) that I answered. This expands on my answer.)

## Optional's Purpose
Optional was introduced to be used in functional programming. It was added because this return expression:

    return Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
      .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
      .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
      .filter(m -> Objects.equals(m.getReturnType(), returnType))
      .findFirst()
      .getOrThrow(() -> new InternalError(...));
is much cleaner than these three lines of code:

    Method matching =
      Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
     .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
     .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
     .filter(m -> Objects.equals(m.getReturnType(), returnType))
     .getFirst();
     if (matching == null)
       throw new InternalError("Enclosing method not found");
     return matching;

(This example comes courtesy of a [blog post by Brian Goetz](http://mail.openjdk.java.net/pipermail/lambda-dev/2012-September/005952.html)) My point is that Optional was written to support functional programming, which was added to Java at the same time. That's the only place where I use it. But gets used a lot of places where it really isn't very helpful, or it's so verbose that it's clumsy. 

## Bad Code Examples

I've seen Optional get used like this, and this is taken from an actual piece of code I encountered at a previous job.

    01 public void someFunction(Widget widget, ... <other parameters>) {
    02   Optional<Widget> widgetOpt = Optional.of(widget);
    03   if (!widgetOpt.isPresent) {
    04     throw new BusinessException("Missing Widget");
    05   }
    06   // ... widgetOpt is never used again
    07   // ... several lines later
    08   processData(Optional.of(widget));
    09   // ...more code
    10 }

What's going on here? One lines 2 and 3, the coder is taking two lines to do a simple null check. Then `widgetOpt` is thrown away, and a second Optional of the same value is created on line 8, because an they call a method that requires it.

The first use is needlessly verbose, and seems to be done just to avoid ever using the `null` keyword in the code, as if it's use is now taboo. But the second use of Optional is more interesting. It's necessary not because they're doing functional programming, but because someone wrote a method that takes an Optional as a parameter.

Why would anyone do this? Probably because Optional is being used to address a flaw in the language design. I call this the Nullness Flaw, and it's responsible for much of the misuse of Java's Optional class.
## The Nullness Flaw
The Nullness Flaw is this: There's no way to specify which of an API's parameters and return values are allowed to be null. It may be mentioned in the javadocs, but most developers don't even write javadocs for their code, and not many will check the javadocs as they write. So this leads to a lot of code like this:

    void someMethod(Widget widget, List<Integer> data) {
      if (widget != null) {
        process(widget, data);
      }
    }
    
This was done everywhere on a past project I worked on, and the ironic thing was that widget couldn't possibly be null here, because it had already been validated the same way by the method that calls this, where it also couldn't be null for the same reason. Once I checked a value in a debugger and found out that it has been repeatedly checked for null nine times up the call stack before ever reaching the method I was debugging. 

This is a very bad practice, because it hides bugs. If `widget` is null here, the method will still fail, but no exception will be thrown. In fact, the purpose of this is to prevent exceptions from ever getting thrown in production code, as if this prevents bugs. (It doesn't.) Java's exception class was written to include a detailed stack trace even when not running in debug mode, which was one of many improvements over C++, but this coding practice throws all this useful information away, just to create the illusion that their code doesn't have bugs. And the customer aren't fooled. They notice when an operation fails to do anything.

I think there was a real thirst to solve this flaw in Java, because so many people saw the new Optional class as a way to add clarity to APIs. This is a laudable goal, but it's not why Optional was created. And it's a bad way to accomplish this. 

## Better Solutions

### IntelliJ Annotations

I have been using a very good solution to this API flaw for several years now, but it's only available to users of the IntelliJ IDE. Intellij provides annotations `@Nullable` and `@NotNull,` which are used with a code inspection that flags variables that violate the annotations' restrictions. So the method above could be written like this:

    void someMethod(@NotNull Widget widget, List<Integer> data) {
        process(widget, data);
    }

The IDE will then check the code that calls this method to make sure the value can't be null, and generates an inspection failure if it gets called incorrectly. IntelliJ also includes a `@Contract` annotation that lets you specify under what conditions a method may return null. I have found this to be a very powerful solution to the problem, but the chief drawback to this is that it only works when using the Intellij IDE. When I'm on a team that has standardized on Eclipse, Netbeans, or some other IDE, I'm the only one who can see the inspection failures.

I've gotten strange pushback against this approach. Most commonly I get the claim that we shouldn't rely on an external tool. I find this a strange argument, since the annotations provide an additional level of checking for our code. So for Eclipse users, there's may be no advantage to using these annotations, but there's no drawback either. Annotations were designed with the understanding that the compiler will ignore any annotations that it doesn't understand, to prevent their use from creating a dependency.

Fortunately, IntelliJ can be configured to use other annotations for the same purpose, so you can set it to use, say, the findBugs annotations, or you can add custom annotations to your project and use those instead.

The strangest pushback I've seen is this: I've been told that this can't possibly work, because Alan Turing proved that it's impossible to determine that a program is correct. This is not what Alan Turning proved. He proved that for any process designed to prove a progam will complete successfully, it is possible to write an algorithm that can defeat your process. When applied to the IntelliJ annotations, this proof simply states that it's possible to write an algorithm that may still have a bug even if the inspection passes. This doesn't mean the inspections will never work, just that they won't catch all possible bugs, which I don't expect them to do anyway.

IntelliJ has a way to declare the annotations externally, so users of Eclipse or Netbeans won't object to them because they never even see them. It's a bit of a pain to set up, but it lets you work around the objections of others, especially when project management has forbidden you to put the annotations in the code. (Yes, this has happened to me.)

### The Nullness Checker

The [Checker Framework](https://checkerframework.org/) provides a Nullness checker that works as an annotation processor, which means it can be used by any IDE, and hence isn't subject to some of the objections raised by the IntelliJ annotations. The Nullness checker (one of several suppported checkers) also uses annotations, called `@NonNull` and `@Nullable`. It doesn't work as well as the IntelliJ system because it doesn't include anything to do what `@Contract` does in IntelliJ. It provides additional annotations for use during object construction, where variables can be temporarily null, but those annotations are hard to use correctly, and the documentation doesn't always have useful examples. But it's more universal than IntelliJ.

I've only used it with new projects. It probably produces a lot of warnings in existing projects written with lots of superfluous tests for null, so it might be tough to retrofit. 

### Kotlin

The cleanest solution may well be to switch to the Kotlin language. Kotlin compiles to Java byte code, so it integrates well with your Java code. But unlike Java, it lets you specify exactly where null values are allowed, and where they're forbidden. In Kotlin, a variable of type Widget can be declared one of three ways:

    var widgetOne: Widget?
    var widgetTwo: Widget = new Widget()
    var widgetThree = new Widget()
Here, the question mark tells the compiler that widgetOne may be null. But widgetTwo and widgetThree are declared to never be null. Also, widgetThree's declaration uses an inferred type — the compiler figures out that the type is a Widget from the constructor call. Now, if the method from the previous example is declared like this:

    fun someMethod(widget: Widget, data: List<Int>) {
      process(widget, data);
    }
The compiler will know that Widget is not allowed to be null, so if I were to pass widgetOne to this method, it would generate a compiler error if I haven't initialized it yet.

Of course, if I have some method where I want to allow a parameter to be null, I can write it like this:

    fun someMethodWithOptional(widget: Widget?, data: List<Int>) {
      ...
    }
    
Kotlin is smart enough to figure out when an optional object is no longer null, so I can say this:

    fun someMethod(widget: Widget? data: List<Int>) {
      if (data != null) {
        someMethod(widget, data)
      } else {
        ...
      }
    }

Kotlin know that widget is no longer null inside the if branch, so it effectively changes the type form Widget? to just plain Widget.

(Kotlin has many other nice features that are beyond the scope of this essay.)

### Conclusion

When facing the Nullness flaw in the Java language, the IntelliJ annotation, the Nullness Checker, and the Kotlin language are all much better solutions than the Optional class. Optional was written for use in Functional programming, and that's the only place I use it. In fact, the java.util.function package is the only one where in the Java API where Optional is used.

#### My rules for Optional

1. Never use it as a method parameter.
2. Never use it as a member of a class.
3. Only use it as a return value if I expect the method to be used in functional programming. 
4. If I expect a value returned by a getter to be used in functional programming, I write two versions of the getter. One returns an Optional, the other doesn't. My two getter methods are named `getXxx` and `optionalXxx`
5. Wherever possible, I design my classes so that members can't possibly be null once the object is constructed. If necessary, I write a Builder class to do the actual construction. (See [Effecive Java](https://www.barnesandnoble.com/w/effective-java-joshua-bloch/1128557432) by Joshua Bloch for a good discussion of Builders.)

I always use one of the three approaches to working around the Nullness flaw. It's a game changer. I love it. My code is much cleaner and more reliable because of it.
