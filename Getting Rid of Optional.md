# Getting Rid of Optional

In my **[Java Optional, Java's Nullness Flaw, and the Kotlin Language](https://github.com/SwingGuy1024/Blog/blob/master/Java%20Misuse%20of%20Optional%20vs%20Kotlin.md)** blog post, I give lots of examples of the misuse of the Optional class. Here I'm going to discuss a case where it's clearly being used correctly, but still (and surprisingly) isn't quite necessary. 

A useful [blog post by Brian Goetz](http://mail.openjdk.java.net/pipermail/lambda-dev/2012-September/005952.html) shows how Optional was intended to get used. Briefly, he says that it's because this:

    return Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
        .stream()
        .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
        .filter(m -> Arrays.equals(m.getParameterTypes(), parameterClasses))
        .filter(m -> Objects.equals(m.getReturnType(), returnType))
        .findFirst()
        .getOrThrow(() -> new InternalError(...));
is cleaner than this:

    Method matching =
        Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
        .stream()
        .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
        .filter(m ->  Arrays.equals(m.getParameterTypes(), parameterClasses))
        .filter(m -> Objects.equals(m.getReturnType(), returnType))
        .getFirst();
    if (matching == null) {
      throw new InternalError("Enclosing method not found")
    }
    return matching;

(In the first case, the `findFirst()` method returns an Optional, which lets us call `getOrThrow(...)` without interrupting the chain.)

This raises a question. Should getters return Optional? It's a valid question, because the Optional return value lets you use them in functional programming, which offers all sorts of advantages, and becoming increasingly common. Functional programming is changing the way software gets written. It's often (but not always) a much cleaner approach, and it has additional benefits like parallel processing without having to write and *very carefully* test any synchronized methods.

But Optional creates a lot of problems. It's often misused, makes the code much more verbose, and will often cloud rather than clarify an API. It's changing the way software gets written in ways that aren't very welcome. (I've written about this in **[Java Optional, Java's Nullness Flaw, and the Kotlin Language](https://github.com/SwingGuy1024/Blog/blob/master/Java%20Misuse%20of%20Optional%20vs%20Kotlin.md)**.)

However, it's actually not necessary to rewrite your getters to return Optional. There's a way to use your getters in Functional code without rewriting them, and it's worth learning, because it will greatly expand the APIs that are available for use in functional code.

To illustrate this, I'd like to bring up an [example from Oracle's documentation](https://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html) on how to take advantage of Optional. 

Consider the following case. You need to model a computer that has an optional sound card, which has an optional USB implementation. Before Optional, you might have written your classes like this:

    public static class Computer {
      private SoundCard soundCard;
      public Computer(SoundCard soundCard) { this.soundCard = soundCard; }
      public SoundCard getSoundCard() { return soundCard; }
    }

    public static class SoundCard {
      private USB usb;
      public SoundCard(USB usb) { this.usb = usb; }
      public USB getUSB() { return usb; }
    }

    public static class USB {
      public USB() { }
      public String getVersion() { return "1.0"; }
    }
  
If you want to get the version of the USB implementation, which might not even exist, you'd have to write a verbose method like this:

    public String getUSBVersion() {
      String version = "UNKNOWN";
      if(computer != null){
        Soundcard soundcard = computer.getSoundcard();
        if(soundcard != null){
          USB usb = soundcard.getUSB();
          if(usb != null){
            version = usb.getVersion();
          }
        }
      }
      return version;
    }

With Optional, you can rewrite your getters and setters, and get somthing like this:

    public static class Computer {
      private SoundCard soundCard;
      public Computer(SoundCard soundCard) { this.soundCard = soundCard;}
      public Optional<SoundCard> getSoundCard() { return Optional.ofNullable(soundCard); } // returns Optional
    }

    public static class SoundCard {
      private USB usb;
      public SoundCard(USB usb) { this.usb = usb; }
      public Optional<USB> getUSB() { return Optional.ofNullable(usb); } // Now returns Optional
    }

    public static class USB {
      public USB() { }
      public String getVersion() { return "1.0"; }
    }

Now, to illutrate the advantages of functional programming (and the use of Optional), you can change your `getUSBVersion()` method to this:

    public static String getUSBVersion(Optional<Computer> computer) {
      return computer
          .flatMap(Computer::getSoundCard)
          .flatMap(SoundCard::getUSB)
          .map(USB::getVersion)
          .orElse("UNKNOWN");
    }

The `Optional.flatMap()` method takes a method that returns Optional, so this only works if two of the getter methods, here specified as function references (`::getSoundCard`, `::getUSB`) return Optional. (The third getter, `USB::getVersion` should NOT return Optional.) This way you don't have to worry about NullPointerExceptions. The functional methods take care of that.

This is certainly a cleaner approach, and changed getters that now return Optional will let you write code this way.

But here's the kicker. You can leave your getters alone. There's a way to write this same functional method without changing your getters. You can do this by adding two static methods to the class, or (better) to a utility class. The two methods here are called `opt()` and `optFn()` for "optional" and "OptionalFunction".

    public static <T> Optional<T> opt(T t) { return Optional.ofNullable(t); }

    public static <T, U> Function<? super T, Optional<U>> optFn(Function<T, U> function) {
      return (Function<T, Optional<U>>) t -> Optional.ofNullable(function.apply(t));
    }

    public static class Computer2 {
      public Computer2(SoundCard2 soundCard) { this.soundCard = soundCard;}
      private SoundCard2 soundCard;
      public SoundCard2 getSoundCard() { return soundCard; }
    }

    public static class SoundCard2 {
      public SoundCard2(USB2 theUsb) { usb = theUsb; }
      private USB2 usb;
      public USB2 getUSB() { return usb; }
    }

    public static class USB2 {
      public USB2() { }
      public String getVersion() { return "2.0"; }
    }

Now your getUSBVersion() method can be written like this:

    public static String getUSBVersion(Computer2 optionalComputer) {
      return opt(computer)
          .flatMap(optFn(Computer2::getSoundCard))
          .flatMap(optFn(SoundCard2::getUSB))
          .map(USB2::getVersion)
          .orElse("UNKNOWN");
    }

Here, I turn `computer` into an Optional by wrapping it in a call to `opt()`, and I use `optFn()` to wrap my two ordinary getters that need to return Optional. In fact, I call the parameter optionalComputer to indicate that it may be null, and the method will still work. (In the earlier version, I could to pass `Optional.empty()` and would return "UNKNOWN", but I couldn't just pass null.)

These two static methods now open up the whole world of ordinary getters to the advangates of functional programming!

So what do I say to the question "Should my getters return Optional?" Well, maybe. If the values might actually be null. But if you just need to use them in functional programming, don't bother. This is a simpler approach.

This raises an interesting question. Should methods like `Optional.flatMap()` been written this way in the first place? I'd have to do some experiments to decide if it would have been a good idea, but it certainly would have changed the landscape of functional programming. 

## Ready for your code

    /**
     * Change your object to a Nullable. This is intended for functional programming when you need an Optional value. 
     * Use this method for brevity. (You could also write {@code Optional.ofNullable(widget)} but {@code opt(widget)} 
     * is quicker.
     * @param t The (possibly null) value to wrap in an Optional
     * @param <T> The type of the value to wrap
     * @return An Optional that wraps the parameter
     * @see #optFn(Function) 
     * @author Miguel Muñoz SwingGuy1024@yahoo.com
     */
    public static <T> Optional<T> opt(T t) { return Optional.ofNullable(t); }
  
    /**
     * optFn is short for OptionalFunction method. Wraps your ordinary getter in a method that returns Optional. 
     * This is intended for functional programming when you need to specify a getter that returns an Optional, 
     * but the getter you need to use doesn't. So, for example, if your getter method is 
     * {@code String Widget.getName()}, and you need to pass it to {@code Optional.flatMap()}, you can say 
     * {@code flatMap(optFn(Widget::getName))}
     * <p>
     * <strong>Example:</strong>  
     * <pre>
     * return opt(computer)
     *   .flatMap(optFn(Computer2::getSoundCard)) // getSoundCard() doesn't return Optional
     *   .flatMap(optFn(SoundCard2::getUSB))      // flatMap() needs a method that returns Optional
     *   .map(USB2::getVersion)
     *   .orElse("UNKNOWN");
     * </pre>
     * @param function The getter or other Function to wrap, usually expressed as a function reference
     * @param <T> The input type of the function (This would be Widget in the example above)
     * @param <U> The return type of the function (This would be String in the example above)
     * @return A function that returns a type of {@code Optional<U>}
     * @see #opt(Object) 
     * @author Miguel Muñoz SwingGuy1024@yahoo.com
     */
    public static <T, U> Function<? super T, Optional<U>> optFn(Function<T, U> function) {
      return (Function<T, Optional<U>>) t -> Optional.ofNullable(function.apply(t));
    }
