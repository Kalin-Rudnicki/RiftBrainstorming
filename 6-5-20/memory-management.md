
Types of memory:
- Stack
- heap

Reasons a chunk of data exists:
- Exclusively for the sake of calculating something else
- To be kept around, and referenced later

---

- The hard part is telling when each of those is the case
- If you return something, it is up to the caller to decide
- When does something need to be reference counted, and when can I tell it will die?
- Desired return address is passed on the stack
    - This allows us to stack allocate something that we want to return
- Variables allocated on the stack that need to have their ref count -- upon exitint stack frame are put in a linked list

```
// Loose layout of the stack

*0 = *1

a
a._1 = ?
a._2 = ?

b
b._1 = a
b._2 = ?
*1 = (b._2, *2)

c
c._1 = a
c._2 = b
c._3 = ?
c._4 = ?
*2 = (c._3, *3)
*3 = (c._4, NULL)
```

- Arbitrarily adding to stack vs re-using
- Custom heap management?

```
heapStack: BArray[Stack[ULong]] = ???

function qAlloc(bytes: UByte) -> ULong =
    heapStack[bytes]
        .sync { stack -> stack.pop } // Option[ULong]
        .getOrElse(alloc(bytes))

function qDealloc(bytes: UByte, addr: ULong) -> Unit =
    heapStack[bytes]
        .sync { stack -> stack.push(addr) }

function alloc(size: ULong) -> ULong = native
```

---

Things to note:
```
// The return address of `+` should be "behind" this function
function a -> Int =
    b + c

// The return address of this should be inside the stack frame of b + c
function b -> Int =
    c + 4

// Same as this
function c -> Int =
    4
```

If the only purpose of the life of an object is to temporarily model data for the purpose of being used somewhere else,
then that should be able to be allocated on the stack.

---

Forward reference pointers:  
A pretty sketchy concept... but I think the compiler should be able to figure it out in a safe way.  
- You can only forward-reference
- You can not forward-use
- Aka, you know where it is, you know something will be there, but you cant use it yet
- This gives us the power to (in a safe way), do things previously only available in low level languages, or pay the price of ugly workarounds

```
// No lookahead, for a simple example
class State private (transitions: Map)

class StateMachine(startState: State) {

    method parse(string: String) -> List[String]? =
        doParse(startState: string.iterator, Nil, Nil)
    
    @TailRec
    private method doParse(state: State, iterator: Iterator[Char], list: List[String], cList: List[Char]) -> List[String]? =
        iterator.next match {
            case Some(c):
                ???
            case None:
                if (state.isFinal)
                    (list + String.fromR(cList)).reverse
                else
                    // "inline-def" of error type
                    // Allows for 
                    throw InvalidEndState(state: state)
        }

    adt ParseError < Error {

        method message -> String =
            this match {
                case 
            }

    }

}

function a -> StateMachine = {
    s0 <- State(
                'A' <& s1
            )
    s1 <- State(
                'A' <& s1,
                'B' <& s2,
                'C' <& s3
            )
    s2 <- State(
                'B' <& s2,
                'C' <& s3,
            )
    s3 <- State(
                'C' <& s3,
            ).finalState

    StateMachine(s0)
}
```
