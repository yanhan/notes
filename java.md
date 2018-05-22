# About

Notes on Java. Covers stuff that I didn't use to know. Some of the stuff is only available from Java 8 onwards.


## Type inference during instantiation of generic classes

An example says it all:

```
List<Integer> nums = new ArrayList<>();
```

Notice that the `Integer` can be omitted in the `ArrayList` constructor.


## Functional Interface

A Functional Interface is an interface which only has one abstract method. It can have any number of default methods.

The cool thing about Functional Interfaces is that they can be constructed using lambda expressions.

Some examples of built in Functional Interfaces:

- [Function<T,R>](https://docs.oracle.com/javase/8/docs/api/java/util/function/Function.html)
- [Predicate<T>](https://docs.oracle.com/javase/8/docs/api/java/util/function/Predicate.html)


## Static Initializers

Executed only once per class initialization and in the order of appearance.

```
public class Student {
    private static int age;

    // Static Initializer
    static {
        age = 5;
    }
}
```


## Multi-catch clause

Catch multiple exception types in one `catch` clause:

```
int x = someFunction();
try {
    if (x > 0) {
        throw new IOException();
    } else {
        throw new NullPointerException();
    }
} catch (IOException | NullPointerException e) {
    e.printStackTrace();
}
```


## User defined annotations

First off, there are 3 types of annotations:

- Marker annotation. These have no parameters
- Single value annotation. These have 1 parameter
- Multivalue annotation. These have more than 1 parameter

To define your own annotation, specify the visibility (most likely `public`) then follow it with `@interface` and then the name of the annotation.

You can define annotations with the same name if they take in a different number of arguments.

```
// Marker annotation
public @interface LogThis {}

// Single value annotation
public @interface LogThis {
    String logFile();
}

// Multivalue annotation
public @interface LogThis {
    String logFile();
    String logLevel() default "INFO";
}
```

To use an annotation, just place it before what is to be annotated

```
@LogThis(logFile="/var/log/myserver.log")
public int calculateCost() {
```


## Generics

|Type parameters|Description|
|---|---|
|`<T>`|Unbounded type; same as `<T extends Object>`|
|`<T,P>`|Unbounded types `T`, `P`; same as `<T extends Object>` and `<P extends Object>`|
|`<T extends P>`|Upper bounded type; a specific type `T` that is a subtype of type `P`|
|`<T extends P & S>`|Upper bounded type; a specific type `T` that is a subtype of type `P` and implements type `S`|
|`<T super P>`|Lower bounded type; a specific type `T` that is a supertype of `P`|
|`<?>`|Unbounded wildcard; any object type, same as `<? extends Object>`|
|`<? extends P>`|Bounded wildcard; some unknown type that is a subtype of type `P`|
|`<? extends P & S>`|Bounded wildcard; some unknown type that is a subtype of type `P` and that implements type `S`|
|`<? super P>`|Lower bounded wildcard; some unknown type that is a supertype of type `P`|

### Using wildcard types

- Use an `extends` wildcard when you only get values out of a data structure
- Use a `super` wildcard when you only put values into a data structure
- Do not use a wildcard if you need to get values out of and put values into a data structure

Some examples will clear things up.

#### Example 1

```
public interface List<E> extends Collection<E> {
    boolean addAll(Collection<? extends E> c);
}

List<Integer> srcList = new ArrayList<>();
srcList.add(0);
srcList.add(1);
srcList.add(2);

// using addAll() method with extends wildcard
List<Integer> destList = new ArrayList<>();
destList.addAll(srcList);
```

In the `addAll` method, we are only getting items out of the the collection `c`, so we use an `extends` wildcard.

#### Example 2

```
public static<T> boolean addAll(Collection<? super T> c, T... elements) { ... }

// using addAll() method with super wildcard
List<Number> sList = new ArrayList<>();
sList.add(0);
Collections.addAll(sList, (byte) 1, (short) 2);
```

In this `addAll` method, since we are only putting items into the collection `c`, we should use a `super` wildcard.


## JShell

A REPL for Java. I have not tried it out but from my reading, here are some useful commands.

- `/imports`: shows the list of libraries currently imported
- `/!`: re-execute the snippet of code that was last run
- `/-<n>`: execute the prior snippet relative to the number supplied, eg. `/-3`
- `/methods`: shows the list of methods currently defined
- `/vars`: shows list of variables currently defined
- `/list`: shows all snippets of code, with integer ids
- `/drop <name or id>`: deletes snippet corresponding to the id or name
- `/edit <name or id>`: edits snippet corresponding to the id or name. Use `/set editor <command>` to choose the text editor to use. Eg. `/set editor vim`
- `/save <filename>`: saves source of all currently active snippets to the filename
- `/reset`: resets JShell's state


## References

- [Java Pocket Guide: Instant Help for Java Programmers](http://a.co/7heCWnu)
