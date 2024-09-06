Brian Goetz wrote this blog post in September of 2007. You can find it here:

https://mail.openjdk.org/pipermail/lambda-dev/2012-September/005952.html

Sometimes this post is unavailable, and may one day disappear. But since I quote from it in my blog post called "Bad Uses of Optional.md,"
I thought I'd make a copy of what Brian Goetz wrote, just in case.

Here's what he wrote, replying to a comment about the intended goal of the Optional class:

>> An alternative of throwing NoSuchElementException was replaced by
>> returning an Optional.
>
> Having Optional enables me to do fluent API thingies like:
>    stream.getFirst().orElseThrow(() -> new MyFancyException())
>
> I would speculate *this* was the intended goal for Optional.

Absolutely.

Take the example from SotL/L from the reflection library.

Old way:

    for (Method m : enclosingInfo.getEnclosingClass().getDeclaredMethods()) {
        if (m.getName().equals(enclosingInfo.getName()) ) {
            Class<?>[] candidateParamClasses = m.getParameterTypes();
            if (candidateParamClasses.length == parameterClasses.length) {
                boolean matches = true;
                for(int i = 0; i < candidateParamClasses.length; i++) {
                    if (!candidateParamClasses[i].equals(parameterClasses[i])) {
                        matches = false;
                        break;
                    }
                }
    
                if (matches) { // finally, check return type
                    if (m.getReturnType().equals(returnType) )
                        return m;
                }
            }
        }
    }
    throw new InternalError("Enclosing method not found");

Without Optional:

    Method matching =
        Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
           .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
           .filter(m -> Arrays.equals(m.getParameterTypes(), parameterClasses))
           .filter(m -> Objects.equals(m.getReturnType(), returnType))
           .getFirst();
        if (matching == null)
            throw new InternalError("Enclosing method not found");
    return matching;

This is much better, but we still have a "garbage varaiable", matching, 
which we have to test and throw before returning.

With Optional:

    return
        Arrays.asList(enclosingInfo.getEnclosingClass().getDeclaredMethods())
            .filter(m -> Objects.equals(m.getName(), enclosingInfo.getName())
            .filter(m -> Arrays.equals(m.getParameterTypes(), parameterClasses))
            .filter(m -> Objects.equals(m.getReturnType(), returnType))
            .findFirst()
            .getOrThrow(() -> new InternalError(...));
