
# Beginning Notes

This is an opinionated take on what I like/dislike about programming languages,
and the current state in my mind of what my "ideal programming language" would be.

---
# Table of Contents

- [Errors](#Errors)  
- [Assertions](#Assertions)  
- [Effects](#Effects)  
- [State](#State)  
- [Dynamic code evaluation](#Dynamic-code-evaluation)  
- [Debugging](#Debugging)  

---
[Table of Contents](#Table-of-Contents)
# Errors

As far as exceptions go, I am a big fan of javas concept of `Checked Exceptions`, where you force the caller to acknowledge that there might be an error.  
That being said, the usability aspect is terrible, and just encourages you to do blanket catches or simply throw the error yourself.  

Then, theres unchecked exceptions, which dont force you to handle it, which is less annoying, but also unsafe,
because you dont know if something might produce an exception, and you arent forced to handle it if it does.  

Lastly, you have the FP approach, where you return something like an Option or an Either.  
I am a huge fan of this, but it ultimately clutters things up, and forces you to do unnecessary amounts of extra handling.  
If you are returning an Option because you actually want to return an Option, thats one thing,
but I am willing to bet that the vast majority of the time, its because you are representing a failed operation, which essentially now needs to be propegated all the way back up the call stack.  

---

So, what is my proposed solution? "Errors", represented by: "?"

```rift
    // This is really modeling divion, but the "/" operator used would violate whats going on here
    function mySub(i1: Int, i2: Int): Int? = // Notice the "?"
        i2 == 0 ? Sub0Err : i1 - i2
```

So, whats going on here? We have essentially modeled the fact that this might error by including the "?" in the return type. Think of the "?" almost like `class MaybeError[E, T]`, where the compiler automatically keeps track of any types of errors you might have accumulated. Then, you have the choice of either pattern matching it to deal with any error, or just including "?" in your own return type.  

Here's here the fun part comes in though, if you have a `T?`, it behaves EXACTLY like a `T`, unless you prefix it with a `~`.

```rift
    res1: Int? = mySub(5, 2) // => 3
    res2: Int? = mySub(5, 0) // => Sub0Err
    res3: Int? = mySub(res1, res2) // => Sub0Err
```

Essentially, what is happening under the hood, is it is automatically able to do map/flatMaps on "?" types, allowing you to never have to worry about them being there.  
Lets look at a quick case (of a terrible example, but just to make the point)

```rift
    function myTerribleFunction1(i1: Int, i2: Int): Int? = {
        mySub(i1, i2)
        i1 + i2
    }
```

Whats going on here? Well, lets look at a few things:
- We do have the "?", signifying an error might have occured
- The error didnt actually affect the return result of the function

This might lead to a programmer blinding adding a "?" because the compiler says an error might have occured, or even worse:

```rift
    function myTerribleFunction2(i1: Int, i2: Int): Int? = {
        iMightCauseADifferentError() // Returns: Unit?
        mySub(i1 + i2)
    }
```

Here, the return type has to be "?", as mySub returns a "?", but off in no-mans-land, the other function might have errored. Whats the solution?  
Any "?" that doesnt affect the return value of the must use the keyword "propegate".

```rift
    function myTerribleFunction3(i1: Int, i2: Int): Int? = {
        propegate ~iMightCauseADifferentError()
        mySub(i1 + i2)
    }
```

`Notice the "~"? Unless we used it, that would just be a normal "Unit", even in the eyes of our special keyword. (TBD)`

A note about what propegate is doing here: if it sees an error, its going to pre-maturely return that error

---
[Table of Contents](#Table-of-Contents)
# Assertions

Now, lets look back at our `mySub` example:

```rift
    function mySub(i1: Int, i2: Int): Int? = // Notice the "?"
        i2 == 0 ? Sub0Err : i1 - i2
```

Now, certainly we can do better...  

Imagine this case here:

```rift
    res: Int? = mySub(5, 2)
```

If I am looking at that, it is so obvious to me that that is not an error.  
`mySub` only fails when `i2 == 0`, and this is clearly not the case.  
It turns out we can do better, and its not even that hard.

```rift
    @Assert(i2 != 0)
    function mySub2(i1: Int, i2: Int): Int? = // Notice the "?"
        i1 - i2
```

So now whats going on here?  
Essentially, you are telling the compiler: When you call `mySub2`, you either guarantee `i2 != 0`, or pay the price of getting a "?".  
Now, going back to what I talked about before, "?" are designed to be as un-annoying to deal with as possible, but you end up with one none-the-less.

Lets look at how this all plays out:

```rift
    res: Int = mySub(5, 2)
```

The compiler can obviously look at the 2, and realize its not 0.

But how about:

```rift
    def test(i: Int): Int? =
        mySub(5, i)
```

How can we get rid of that pesky "?" ?  
The answer is simple:

```rift
    @Assert(i != 0)
    def test(i: Int): Int =
        mySub(5, i)
```

Now, whoever calls you either provides that guarantee, or pays the price themself.

I imagine at the beginning, these assuptions will be VERY basic, but I would eventually like to take this sort of thing to extremes,
like being able to track how many items are in a list, or knowing for sure that a map has a certain value (and therefore not needing an Option[T])

Now, lets look at one last little trick we can do here:

```rift
    res: Int? = mySub(5, 0)
```

What happens here? We fail to compile.  
If we know some sort of guaratee about our program, and we know for sure the assertion will fail, why on earth would we let it compile?  

---
[Table of Contents](#Table-of-Contents)
# Effects

Lucky for the reader, this section is much simpler than the one above.

If you are familiar with FP, you are very familiar with what an effect is. Here, we model them with "!".  
Heres what it looks like:

```rift
    def myEffectfulFunction: Unit! = // ...
```

And another very simple concept that we borrow from "?": you have it, you pass it:

```rift
    def myOtherFunction: Unit! =
        myEffectfulFunction // This has a "!", so we have to be as well
```

"!" has a similar concept to "propegate" as well, "acknowledge".  
Any effectful part of the function that doesnt directly impact the result of the function/method must be acknowledged.

```rift
    @Assert(i2 != 0) // Why is this line here?
    def myTerribleFunction(i1: Int, i2: Int): Int! = {
        res: Int = mySub(i1, i2)
        acknowledge println("Result: $res")
        res
    }
```

`If you have a "?!", should "propegate" serve the purpose of acknowledging the "!" ?`

---
[Table of Contents](#Table-of-Contents)
# State

Now, this one is still a little up in the air, as I havent full figured it out yet, but state will use something similar, and we use "$" or "#"

Variable names that end in "$" are non-final, or "vars" from scala.  
Types that end in "$" are mutable.  

```rift
    person: Person = Person("first", "last")
    person.lastName = "last2" // compile error, person is not mutable
```
```rift
    person$: Person = Person("first", "last")
    person.lastName = "last2" // compile error, person is not mutable (just cuz it can be set to a different value doesnt mean its mutable)
```
`Im not sure if I want to actually include the "$" in the variable name, eg: "person" or "person$" when referencing it?`
```rift
    person: Person$ = Person("first", "last")
    person.lastName = "last2" // Now this works
    person = Person("other", "person") // compile error, person is not a variable (just cuz it is mutable doesnt mean you can set it to something else)
```
```rift
    person$: Person$ = Person("first", "last")
    person.lastName = "last2"
    person = Person("other", "person") // Now we can do both
```

"$" is used in param list to say "I am going to change this thing, you need to give me something mutable"

```rift
def test(person$: Person): Unit =
    person.lastName = "changed" // compile error, can you tell why?
```
```rift
def test(person: Person$): Unit =
    person.lastName = "changed" // Now thats better
```

"#" is used in reference to undecided mutability.  
`I have not yet decided if I want "#" to represent decided or undecided, and an absence of "#" would represent the other`  
The prime example of something that returns something of undecided mutability is a constructor.  
It doesnt care whether you change it or not, its sole purpose is to provide you with a value.  

I have a lot more thinking to do on this subject, but I would think that almost anything thats sole purpose is to take things in and give you a result,
that the result should be undecidedly mutable.  
That being said, if at some point, the function you called promised someone else that it was immutable, then you have to follow that as well.  

This is one of the things Im still up in the air about, but I think there needs to be a distinction here about, but I think the biggest distinction is whether or not you decide to save the value.  
`This would need some thinking when multithreading comes into play, but...` if I am not going to change something, why do I necesarrily care if someone else will? I dont necessarily need something thats immutable, and I dont need to enforce to everyone that calls me, that that thing needs to be immutable for its entire lifetime.  
I think the whole threading thing wouldnt be that complicated, I think it just comes down to when you give something access in a thread, you are forced to decide whether you want it to be mutable or immutable. Also, I think you need to decide one or the other for global variables.

Now, it is an important distintion to make that having "undecidedly mutability" doesnt necessarily make you mutable. Its simply saying "I dont care if this is mutable or not".  
As soon as something that is undecidedly mutable gets assigned/passed to something that is explicitly mutable/immutable, it is forced to hold that constraint as well.

---
[Table of Contents](#Table-of-Contents)
# Dynamic code evaluation

Being able to do dynamic code evaluation has always been something that is very important to me. I love my meta-programming. I would ideally like to get to the point where you can dynamically define "dynamic classes" and do all sorts of fun stuff with it, but I think that is a stretch goal for way down the line.  
A medium goal would be the ability to eval strings.  
A good start would be able to do things similar to java reflection.

Have you ever used javas reflection, its terrible... so much exception handling, is just so... oh wait...  
We have a SUPER nice way to handle these sort of errors, remember?

Lets take a look at a basic example:
The `..` operator signals we are going to be making dynamic calls, and anything in that expression becomes dynamic as well.  

```rift
person: Person = Person("first", "last")

res1: Any? = person..lastName // => "last"
res2: String? = person..lastName.cast[String] // => "last"
res3: Int? = person..lastName.cast[Int] // Error
res4: Any? = person..methodIDontHave // Error

res5: String? = person..lastName.cast[String].capitalize // => "LAST"
```

---
[Table of Contents](#Table-of-Contents)
# Debugging

I think it will be important to have some sort of "debug" (probably with a more unique name).  
The reason for this, is with so much of the forced "?" and "!" handling, if you were to throw in a print just for debugging purposes,
you would then be forced to do all sorts of passing "!" up the stack trace, just to have to go and delete it soon.  
Then, if you compile with something like `--debug=on`, it will include the debugging stuff, and if you compile without that, anything that is debugging will be omitted.  
An important note is that if you encounter a "?" in something that is debugging, the program will exit.
