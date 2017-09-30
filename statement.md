# Kotlin on the JVM - How can it provide so many features?

_Disclaimer: My articles are published under 
<a href="https://creativecommons.org/licenses/by-nc-nd/4.0/legalcode" target="_blank">"Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)"</a>._

© Copyright: Simon Wirtz, 2017
https://blog.simon-wirtz.de/kotlin-on-the-jvm-byte-code-generation/

Feel free to share.

## Introduction

What exactly is a *"JVM language"*? Isn't only Java meant to be run on the JVM?
<a target="_blank" href="http://kotlinlang.org">Kotlin</a> provides many features which aren't available in Java such as a proper _function type_, _extension functions_ or _data classes_.
How is this even possible? 
I've taken a deeper look at how Kotlin is made possible and what "JVM language" actually means. I hope to help some people understanding a few things better :)

For a more detailed introduction to _Kotlin's features_ you can have a look at my recent posts like <a target="_blank" href="https://blog.simon-wirtz.de/setup-vert-x-application-written-in-kotlin-gradle-build/">this one</a>.

## The Java Virtual Machine
_A quick and simple definition_: The <a href="https://en.wikipedia.org/wiki/Java_virtual_machine">Java Virtual Machine</a> is used by computers to run *Java bytecode*. 
Actually there's a lot more to learn about this complex tool, which is described very deeply in <a href="https://docs.oracle.com/javase/specs/jvms/se7/html/index.html" target="_blank">Oracle's Spec</a>.
As you might already know, the JVM is an abstract virtual computer running on various operating systems.
In fact, the JVM is what makes Java "platform independent", because it acts as an abstraction between the executed code and the OS.
Just like any real computer, the JVM provides a defined _set of <a href="https://simple.wikipedia.org/wiki/Instruction_(computer_science)">Instructions</a>_ which can be used by a program and are translated to machine specific instructions by the JVM itself later on.

As described in the  <a href="https://docs.oracle.com/javase/specs/jvms/se7/html/index.html" target="_blank">JVM Spec</a>, the Java Virtual Machine _doesn't know anything_ about *the programming language Java*. However, it defines the binary format `class` which is a file containing machine instructions (bytecodes) to be executed beside some more information. This is a very interesting point, because it actually means, that:


1. _the JVM isn't only dedicated to Java as a programming language._
2. _you are free to choose a technology for creating JVM programs as long as you provide proper `class` files that are compliant to the very strict constraints._
3. _regardless of programming languages, any Java bytecode can interoperate with other Java bytecode on the JVM._

## Creation of Class Files

The process of creating class files from human-readable source code is what a _compiler_ does. One example is Oracle's Java Compiler shipped with the <a href="http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html" target="blank">JDK</a> (`javac`) that is capable of compiling `.java` files to `.class` files. 
In Addition to Java, many other <a href="https://en.wikipedia.org/wiki/List_of_JVM_languages" target="_blank">JVM languages</a> have emerged in the last few years, which are supposed to provide an alternative abstraction for us developers to create programs for the JVM.
One of these languages is <a target="_blank" href="https://kotlinlang.org/docs/reference/">*Kotlin*</a>.

## Kotlin Bytecode Generation, Version 1.1

As stated in the official <a href="https://kotlinlang.org/docs/reference/faq.html" target="_blank">FAQs</a> "Kotlin produces Java compatible bytecode", which means that the Kotlin compiler is capable of transforming all the nice features into JVM compatible instructions and this can even be observed using <a href="https://www.jetbrains.com/idea/" target="_blank">IntelliJ IDEA</a> tools.  
Let's look at some examples:

```kotlin
//File.kt
fun foobar(){}
```

This simple top level function defined in a .kt file can be investigated with IntelliJ: <br/>
*"Tools → Kotlin → Show Kotlin Bytecode"* will open a new window inside the IDE providing a live preview of the Java bytecode the compiler would create for the current .kt file.

```java
public final class de/swirtz/kotlin/FileKt {
  // access flags 0x19
  public final static foobar()V
   L0
    LINENUMBER 3 L0
    RETURN
   L1
    MAXSTACK = 0
    MAXLOCALS = 0

  @Lkotlin/Metadata;
  // compiled from: File.kt
}
```

I'm afraid only a few people can actually read these files, which is why we can also choose the option "Decompile". Afterwards we'll be presented a Java class representing the functionality previously described with Kotlin:

```java
public final class FileKt {
   public static final void foobar() {
   }
}
```

As you can see and probably already know, a Kotlin top level class is compiled into a final Java class with a static function (This structure looks like what extension functions mean to replace: utility classes). Let's see a more difficult one:


```kotlin
class MyClass(val i: Int)

fun MyClass.myExtension(value: String) = value.length
```

This one shows a simple class `MyClass` with a `property` of type `Int` as well as a top level extension function.
First we should have a look at what the class is compiled to, which is quite interesting as we used a _primary constructor_ and the _val keyword_ here.

```java
public final class MyClass {
   private final int i;

   public final int getI() {
      return this.i;
   }

   public MyClass(int i) {
      this.i = i;
   }
}
```

As we would expect: the `property` is a `final` member being assigned in the single constructor. Yet, so much simpler in Kotlin :)


```java
public final class FileKt {
   public static final int myExtension(@NotNull MyClass $receiver, @NotNull String value) {
      Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
      Intrinsics.checkParameterIsNotNull(value, "value");
      return value.length();
   }
}
```

The extension function itself is compiled to a static method with its _receiver object_ as a parameter in the Java code.
One thing we can also observe in the example is the use of a class called `Intrinsics`. This one is part of the Kotlin <a href="https://kotlinlang.org/api/latest/jvm/stdlib/index.html" target="_blank">stdlib</a> and is used because the parameters are required to be not `null`. <br/>
Let's see what would happen if we changed the inital extension function's parameter to `value: String?` and of course access `length` in a safe way.

```java
----
public final class FileKt {
   @Nullable
   public static final Integer myExtension(@NotNull MyClass $receiver, @Nullable String value) {
      Intrinsics.checkParameterIsNotNull($receiver, "$receiver");
      return value != null?Integer.valueOf(value.length()):null;
   }
}
```

Checking `value` is not necessary anymore since we told the compiler that `null` is an acceptable thing to point to. <br/>
The next example is a bit more tricky. It's the one with the greatest difference between Kotlin and Java code:

```kotlin
----
fun loopWithRange(){
    for(i in 5 downTo 1 step 2){
        print(i)
    }
}
```

```java
 public static final void loopWithRange() {
      IntProgression var10000 = RangesKt.step(RangesKt.downTo(5, 1), 2);
      int i = var10000.getFirst(); //i: 5
      int var1 = var10000.getLast(); //var1: 1
      int var2 = var10000.getStep(); //var2: -2
      if(var2 > 0) {
         if(i > var1) {
            return;
         }
      } else if(i < var1) {
         return;
      }

      while(true) {
         System.out.print(i);
         if(i == var1) {
            return;
         }

         i += var2;
      }
   }
```

Although the Java code is still quite understandable, probably nobody would write it in real life, because a simple `for` could do it, too. We need to consider that `downTo` and `step` are _infix notations_, which are function calls actually.
In order to provide this flexibility, a little more code seems to be necessary.

What do you think? Doesn't like nice although the Kotlin code is brilliant, right?

## Conclusion

I think, most of the time you don't really care about what the Kotlin compiler produces for us. Yet, I find observing it really interesting and helpful as it supports answering my initial questions in some way. Of course, Kotlin is much more than just abstracting Java's operators since it also provides so many extensions to existing Java classes like `List` or `String`.
Nevertheless, we also saw, that sometimes the compiled Java code is more verbose as it had to be. Could this impact performance? Yes indeed, it does have minor effects. Have a look at this <a href="https://de.slideshare.net/intelliyole/kotlin-bytecode-generation-and-runtime-performance" target="_blank">presentation</a> by Dmitry Jemerov if you're interested in more "Kotlin -> Java bytecode" examples considering perfomance, too. 

Let me know what you think about it and get in touch if you like!

Simon