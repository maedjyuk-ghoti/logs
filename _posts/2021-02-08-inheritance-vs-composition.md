---
layout: post
title: Inheritance vs Composition
date: 2021-02-08 22:58:29 -0400
categories: programming oop
---

# Introduction
The cornerstone of Object Oriented Programming (OOP) is inheritance. It's the biggest talking point within the paradigm. This is not an entry to discuss what is and is not OOP. Here, we will discuss Inheritance vs Composition for writing OOP in the context of Kotlin.

# Definitions
## Inheritance
["In object-oriented programming, inheritance is the mechanism of basing an object or class upon another object (prototype-based inheritance) or class (class-based inheritance), retaining similar implementation."][0]

## Composition
["In computer science, object composition is a way to combine objects or data types into more complex ones."][1]

# Standing on the Shoulders of Giants
## Gang of Four
(insert quote here)
## Joshua Bloc
In Effective Java, Joshua Bloc writes "Item 16: Prefer composition over inheritance".

# Basic Examples
## Fragile Base Class
The common example given at this point is the `CountingHashSet` or `CountingArrayList`. Really any `CountingCollection`.
```
class CountingHashSet<E> : HashSet<E>() {
    var counter: Int = 0
        private set

    override fun add(element: E): Boolean {
        counter++
        return super.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        counter += elements.size
        return super.addAll(elements)
    }
}
```
And then you are asked to evaluate what the output will be for the following code:
```
val s = CountingHashSet<String>()
s.addAll(listOf("A","B","C"))
println("${s.counter}")
```
Did you get `3`? Is it possible to get anything else? What about `6`? Did you know that `HashSet` is implementing `addAll(...)` by calling `add(...)` multiple times? Because it is. And you overrode it and so your `add(...)` is being called and you're double counting.

## Leaking APIs
Consider the following:
```
class MyList : ArrayList<Int>() {
    override fun add(element: Int): Boolean {
        return super.add(element)
    }

    override fun addAll(elements: Collection<Int>): Boolean {
        return super.addAll(elements)
    }
}
```
What can you do with this class? Well, you can `add` and `addAll` of course. Did you also know you can do anything that `List` allows? The entirety of `List` is exposed for use.
```
val myList = MyList()
myList.size
myList.clear()
myList.remove(0)
myList.contains(1)
```

## The Fix
The fix for both of these issues is not to use inheritance, but composition. (note: these will not be perfect and will only address the issues brought up in their sections)
```
class CountingHashSet<E>(private val hs: HashSet<E>) {
    var counter: Int = hs.size
        private set

    override fun add(element: E): Boolean {
        counter++
        return hs.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        counter += elements.size
        return hs.addAll(elements)
    }
}
```
and
```
class MyList(private val al: ArrayList<Int>) {
    fun add(element: Int): Boolean {
        return al.add(element)
    }

    fun addAll(elements: Collection<Int>): Boolean {
        return al.addAll(elements)
    }
}
```

# Going Further
## 2 types of inheritance
It should be noted that there are 2 types of inheritance. The Basic Examples section only dealt with class, or implementation, inheritance which is characterized with the use of `extends` in Java. In Kotlin, it is characterized by calling the constuctor `()` in the class definition.
```
// Java
public class MyArrayList<E> extends ArrayList<E> {...}

// Kotlin
class MyArrayList<E> : ArrayList<E>() {...}
```
Interface inheritance is characterized by the `implements` keyword in Java and no constructor call in Kotlin.
```
// Java
public class MyList<E> implements List<E> {...}

// Kotlin
class MyList<E> : List<E> {...}
```
Interface inheritance is prefered over implementation inheritance. In this way none of the baggage of an implementation is kept. In this article, inheritance refers to implementation inheritance unless otherwise specificed

## Flexibility
On its own, preferring Composition over Inheritance does not grant flexibility to design.
```
class MyArrayList<E> : ArrayList<E> {...}
class MyArrayList<E>(private val al: ArrayList<E>) {...}
```
Though it could be argued that the composition variation allows any subclass of ArrayList, there is no appreciable difference in flexibility here as anything passed in would be inheriting all the details of ArrayList. Instead, if we prefer the least restrictive option that suits our goals, we can plug in different implementations as needed.
```
1: class MyClass<E>(private val l: List<E>) {...}
2: class MyClass<E>(private val c: Collection<E>) {...}
3: class MyClass<E>(private val i: Iterable<E>) {...}
```
In this manner:
* 1 can take an ArrayList or a LinkedList just as easiliy.
* 2 can take any list or a map or a set or array, etc.
* 3 can take so much more.

Pick what is appropriate for your use. If you require a list over a set, use option 1. If you want a map, specificy a map. But think hard about whether your use requires a hashmap or not before specifying it.

## Eat Elsewhere
I like books. The following quote comes from a book. But not a programming book.
```
"What are the three most important rules of the chemist?"
"Label clearly. Measure twice. Eat elsewhere."
- Patrick Rothfuss, The Name of the Wind
```
There is probably a better word here (something less full of baggage than Dependency Injection) but this quote expresses the point quite well enough. Don't create and use in the same place or you lose a lot of flexibility.

What are the differences between the following?
```
class MyClass<E> {
    private val l: List<E> = arrayListOf<E>()
    ...
}
val mc = MyClass<Int>()
```
and
```
class MyClass<E>(private val l: List<E>) {...}
val mc = MyClass<Int>(arrayListOf<Int>())
```
The second class is infinitely more flexible _at runtime_. Sure you can go in and change the first example to a linkedlist if you prefer but you don't have to edit the second one at all for that to work. Just change what you pass into it. All of the examples so far have followed this principle.

Though not a part of the inheritance vs composition debate, it is a lead into the next section.

# Testing
Taking all of the above, from the Basic Examples and Going Further sections, we finally get to what I believe is the absolute strongest argument in favor of Composition over Inheritance: Testing.

When you inherit an implementation you inherit and all of its implementation details _and_ any flaws. This can make testing a headache where you end up partially debugging the base class. This quickly turns into a nightmare when you can't fix the base class because it's provided by a framework or library.

Let's revist the first example and talk through some ideas.
```
class CountingCollection<E> : HashSet<E>() {
    var counter: Int = 0
        private set

    override fun add(element: E): Boolean {
        counter++
        return super.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        counter += elements.size
        return super.addAll(elements)
    }
}
```
Let's look at a test for `CountingCollection<E>.add(...)`:
```
@Test
fun `test add(1)` {
    val cc = CountingCollection<String>()
    assertTrue(cc.add("Hello"))
    assertEqual(1, cc.counter)
}
```
So far so good. How about `CountingCollection<E>.addAll(...)`:
```
@Test
fun `test addAll(1)` {
    val cc = CountingCollection<String>()
    assertTrue(cc.addAll(listOf("Hello", "Hello")))
    assertEqual(2, cc.counter)
}
```
Oh no! This test will fail. Can you guess why? Did you guess that it's because the underlying `HashSet<E>.addAll(...)` returns false when the same string is added twice? Awesome! So we just change the test... wait! Are we testing our extension or `HashSet<E>`? Using only composition wouldn't fix this.
```
class CountingCollection<E> {
    private val hs: HashSet<E> = hashSetOf<E>()
    var counter: Int = 0
        private set
    ...
}
```
It has all of the same flaws as far as testing is concerned. Even loosening the restrictive class type doesn't help.
```
class CountingHashSet<E> {
    private val hs: Collection<E> = hashSetOf<E>()
    ...
}
```

Let's try the composition version:
```
class CountingCollection<E>(private val hs: Collection<E>) {
    var counter: Int = hs.size
        private set

    override fun add(element: E): Boolean {
        counter++
        return hs.add(element)
    }

    override fun addAll(elements: Collection<E>): Boolean {
        counter += elements.size
        return hs.addAll(elements)
    }
}
```
And the tests:
```
private val mockCollection: Collection<String> = mock()

@Test
fun `test add(1)`() {
    `when`(mockCollection.add(any())).thenReturn(true)
    val cc = CountingCollection<String>(mockCollection)
    assertTrue(cc.add("Hello"))
    assertEqual(1, cc.counter)
}

@Test
fun `test addAll(1)`() {
    `when`(mockCollection.addAll(any())).thenReturn(true)
    val cc = CountingCollection<String>(mockCollection)
    assertTrue(cc.addAll(listOf("Hello", "Hello")))
    assertEqual(2, cc.counter)
}
```
Now the collection backing our `CountingCollection` class is completely separate. This allows us to test under a variety of circumstances. Even seeding it with data to test against
```
@Test
fun `test addAll(1)`() {
    val testSet = setOf("foo", "bar")
    val cc = CountingCollection<String>(testSet)
    assertTrue(cc.addAll(listOf("Hello", "World")))
    assertEqual(4, cc.counter)
}
```

# Summary
Prefer composition over inheritance to avoid
* Fragile Base Classes
* Leaking APIs
* Inflexible Hierarchy
* Messy Tests

# Sources
## References
0 - [Inheritance Wiki][0]  
1 - [Composition Wiki][1]  

## Extra Reading
[Extends is Evil][2]  
[Composition over Inheritance Wiki][3]

<!-- source reference links -->
[0]: https://en.wikipedia.org/wiki/Inheritance_(object-oriented_programming) "Inheritance Wiki"
[1]: https://en.wikipedia.org/wiki/Object_composition "Composition Wiki"
[2]: https://www.infoworld.com/article/2073649/why-extends-is-evil.html
[3]: https://en.wikipedia.org/wiki/Composition_over_inheritance
