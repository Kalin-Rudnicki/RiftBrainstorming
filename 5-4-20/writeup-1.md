
# Beginning Notes

This is an opinionated take on what I like/dislike about programming languages,
and the current state in my mind of what my "ideal programming language" would be.

---
# Main Points

`TODO : Links`

---
# Errors ("Exceptions")

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
        i2 == 0 ? Div0Err : i1 - i2

```

So, whats going on here? We have essentially modeled the fact that this might error by including the "?" in the return type. Think of the "?" almost like `class MaybeError[E, T]`, where the compiler automatically keeps track of any types of errors you might have accumulated. Then, you have the choice of either pattern matching it to deal with any error, or just including "?" in your own return type.  

Here's here the fun part comes in though, if you have a `T?`, it behaves EXACTLY like a `T`, unless you prefix it with a `~`.

```rift

    res1: Int? = mySub(5, 2) // => 3
    res2: Int? = mySub(5, 0) // => Div0Err
    res3: Int? = mySub(res1, res2) // => Div0Err

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

`Notice the "~"? Unless we used it, that would just be a normal "Unit", even in our special keywords eyes. (TBD)`
