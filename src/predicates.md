# Predicates

Predicates give a name to a parameterized assertion. A predicate can have any number of parameters, and its body can be any self-framing Gobra assertion using only these parameters as variable names. Predicate  definitions can be recursive, allowing them to denote permission to and properties of recursive heap structures such as linked lists and trees.

The following snippet defines a type `node`, representing a node of a linked list. The predicate `list`, parameterized by `ptr`, abstracts the permissions of a linked list. The predicate body contains permissions to the `value` and `next` fields of the head and, recursively, contains permissions to the rest of the list if the tail is not empty, that is, if `ptr.next != nil`.

```go
package tutorial

type node struct {
  value int
  next *node
}

pred list(ptr *node) {
  acc(&ptr.value) && acc(&ptr.next) && (ptr.next != nil ==> list(ptr.next))
}
```

Predicate instances can occur in function specifications. In Gobra, predicates are *iso-recursive*, meaning that predicate instances are not equivalent to their body. As such, the following program does not verify:

```go
requires list(ptr)
ensures  list(ptr)
func testPredFail(ptr * node) {
  assert list(ptr) // succeeds
  assert acc(&ptr.value) && acc(&ptr.next) // fails
}
```

To make the snippet verify, Gobra requires an additional `unfold` annotation, telling Gobra to replace a predicate instance with its body. Conversely, the `fold` annotation exchanges the predicate body with the predicate instance. Using `unfold`, the adapted version of the previous snippet verifies:

```go
requires list(ptr)
ensures  list(ptr)
func testPred(ptr * node) {
  assert list(ptr)
  unfold list(ptr) // exchanges list(ptr) with its body
  assert acc(&ptr.value) && acc(&ptr.next) 
  fold list(ptr)   // folding back the predicate for the postcondition
}
```

In cases where statements are not allowed (e.g., specifications and bodies of pure functions), we use `unfolding` expressions, which temporarily unfold and fold a predicate instance. For instance, in the body of the `header` function below, the only way to acquire access to `ptr.value` is by `unfolding list(ptr)`.

```go
requires list(ptr)
pure func head(ptr *node) int {
  return unfolding list(ptr) in ptr.value
}
```

In the previous examples of this section, the precondition requires write permissions to the list. If a function does not modify the list, write permissions are stronger than required. To specify that we want less than write permissions, we use accessibility predicates, similar to heap locations. In the following example, the argument `p` of type `perm` specifies the permission amount we use for the predicate. The precondition `p > 0` asserts that this permissions amount is positive and, thus, grants read access to the elements of the list.

```go
requires p > 0
requires acc(list(ptr), p)
pure func (ptr *node) contains(value int, ghost p perm) bool {
  return unfolding list(ptr) in ptr.value == value || (ptr.next != nil && ptr.next.contains(value, p))
}
```

Often, in particular for pure functions, we do not care about the exact permission amount since all permissions in the precondition are returned implicitly in the postcondition. In these cases, we can use a wildcard permission amount `_`specifying an arbitrary positive permission amount.

```go
requires acc(list(ptr), _)
pure func (ptr *node) contains(value int) bool {
  return unfolding list(ptr) in ptr.value == value || (ptr.next != nil && ptr.next.contains(value))
}
```

Gobra predicates can also be abstract, i.e., have no body, to represent opaque permissions. Abstract predicates cannot be unfolded.


## Ghost Code

Often, introducing additional state, such as additional variables or arguments is useful for specification and verification. Consider the following `contains` function. The function returns whether or not an element `x` is contained in a slice `s`. One way to express that an element is contained in a slice is to specify that there is an index within the slice at which the element is stored. In the example, we use the ghost out-parameter `idx`, annotated with the keyword `ghost`, to store this index, allowing us to write an appropiate postcondition.

```go
package tutorial

requires forall k int :: 0 <= k && k < len(s) ==> acc(&s[k], 1/2)
ensures  forall k int :: 0 <= k && k < len(s) ==> acc(&s[k], 1/2)
ensures  isContained ==> 0 <= idx && idx < len(s) && s[idx] == x
func contains(s []int, x int) (isContained bool, ghost idx int) {

  invariant 0 <= i && i <= len(s)
  invariant forall k int :: 0 <= k && k < len(s) ==> acc(&s[k], 1/4)
  for i := 0; i < len(s); i += 1 {
    if s[i] == x {
      return true, i
    }
  }

  return false, 0
}
```

Apart from the ghost in and out-parameters, Gobra offers additional ghost constructs: ghost variables, ghost types, ghost statements, and ghost methods and functions. A ghost variable is defined via a ghost declaration. For instance, the statement`ghost var x int = 42` declares a ghost variable `x` and assigns it 42. Ghost types such as sequences, sets, and multisets are mathematical types useful for specification purposes (see the [datatype section](#Datatypes) for more details).  Ghost if-then-else and loops are used to perform case splits and inductions, respectively. To make an actual statement a ghost statement, just add the `ghost` keyword before the statement. Pure ghost functions can be used to abstract heap state to a mathematical representation. The `toSeq` function below takes as parameter a slice and returns a mathematical sequence containing the elements of the slice. A ghost function or method is annotated with the keyword `ghost`, which at this point, has to be written before the pre and postconditions. For ghost functions and methods, all in- and out-parameters are implicitly ghost. Gobra checks that no ghost code influences actual code. Note that the opposite does not have to hold. For instance, the value of an actual variable can be assigned to a ghost variable.

```go
ghost
requires forall j int :: 0 <= j && j < len(s) ==> acc(&s[j],_)
ensures len(res) == len(s)
ensures forall j int :: {s[j]} {res[j]} 0 <= j && j < len(s) ==> s[j] == res[j]
pure func toSeq(s []int) (res seq[int]) {
  return (len(s) == 0 ? seq[int]{} :
      toSeq(s[:len(s)-1]) ++ seq[int]{s[len(s) - 1]})
}
```

## Datatypes

Gobra supports many of Go's native types, namely integers (`int`, `int8`, `int16`, `int32`, `int64`, `byte`, `uint8`, `rune`, `uint16`, `uint32`, `uint64`, `uintptr`), strings, structs, pointers, arrays, slices, interfaces, and channels. Note that currently the support for strings and specific types of integers such as `rune` is very limited. 

In addition, Gobra introduces additional ghost types for specification purposes. These are sequences (`seq[T]`), sets (`set[T]`), multisets (`mset[T]`), and permission amounts (`perm`). Gobra supports their common operations: sequence concatenation (`seq1 ++ seq2`), set union (`set1 union set2`), membership (`x in set1`), multiplicity (`x # set1`), sequence length (`len(seq1)`), and set cardinality (`len(set1)`). 


## Interfaces

In Go, interfaces consist of method signatures. In Gobra, all these method signatures additionally have a method specification. Furthermore, an interface can define predicates to abstract over the heap structure of the underlying dynamic value. Consider the following declaration for an interface `stream`. The interface has two methods, the pure method `hasNext`, returning whether more elements are available, and the method `next`, returning the next element of the stream. The predicate `mem` abstracts over the underlying heap structure.

<table>
<tr>
<td> Go code </td> <td> Gobra code </td>
</tr>
<tr>
<td> 

```go
package tutorial

type stream interface {


  
  hasNext() bool

  
  
  next() interface{}
}
```

</td>
<td>

```go
package tutorial

type stream interface {
  pred mem()

  requires acc(mem(), 1/2)
  pure hasNext() bool

  requires mem() && hasNext()
  ensures  mem()
  next() interface{}
}
```

</td>
</tr>
</table>

The following snippet illustrates an implementation of the `stream` interface. The function `typeOf` returns the dynamic type of an interface. The type assertion `y.(int)` requires that the interface `y` has the dynamic type `int` and returns the dynamic value of the interface. Note that without a preceding `typeOf(y) == int` the type assertion fails.

```go
// implementation
type counter struct{ f, max int }

requires acc(x, 1/2)
pure func (x *counter) hasNext() bool {
  return x.f < x.max
}

requires acc(&x.f) && acc(&x.max, 1/2) 
requires x.hasNext()
ensures  acc(&x.f) && acc(&x.max, 1/2) && x.f == old(x.f)+1 
ensures  typeOf(y) == int && y.(int) == old(x.f)
func (x *counter) next() (y interface{}) {
  y = x.f
  x.f += 1
  return
}
```

Before a value of type `*counter` can be assigned to the `stream` interface, one has to show that `*counter` is a behavioral subtype of `stream`. This means that for all methods the precondition of the interface method implies the precondition of the implementation method and the postcondition of the implementation method implies the postcondition of the interface method.  In Gobra, such proofs are done via *implementation proofs*. The snippet below shows such an implementation proof for `*counter` and `stream`. The definition of the predicate `mem` is part of the implementation proof and defines the permissions that `*counter` requires to satisfy the interface specification. The implementation proof clause is denoted with the keyword `implements`. Inside the implementation proof clause, for each method of the interface, there is a proof method. The pre and postconditon of the proof method are omitted as they are determined by the corresponding method of the interface. Implementation proofs are heavily syntactically restricted. The body must have a single call to the method of the implementation with the right receiver and the in- and out-parameters in the correct order. Additionally, the body is only allowed to contain assert statements and fold and unfold annotations to manipulate predicate instances. In the example, we require unfolds and folds to shift from the `mem()` predicate instance of the interface methods to the direct pointer permissions of the implementation methods and vice versa. At a later point in time, additional proof annotations will be allowed.


```go
// implementation proof
pred (x *counter) mem() { acc(x) }

(*counter) implements stream {
  pure (x *counter) hasNext() bool {
    return unfolding acc(x.mem(), 1/2) in x.hasNext()
  }

  (x *counter) next() (res interface{}) {
    unfold x.mem()
    res = x.next()
    fold x.mem()
  }
}
```

If an implementation proof or part of it is omitted, Gobra tries to infer the missing parts. However, at this point in time, Gobra uses a simple heuristic that fails if the proof requires folding and unfolding of predicates. As a special case, the empty interface `interface{}` is trivially implemented by all types. As such, in `*counter.next`, the statement `y = x.f` assigning an integer to the empty interface does not require a manual implementation proof.

The snippet below shows a small client that instantiates a `*counter`, assigns it to the `stream` interface, and calls the method `next`. Note that Gobra can show that the precondition of `y.next()`, namely `y.hasNext()` holds.

```go
// client code
func client() {
  x := &counter{0, 50}
  var y stream = x
  fold y.mem()
  var z interface{} = y.next()
}
```

Implementation proofs can be placed in a different package than the orginal type definition. In this case, the syntax for the predicate is slightly different. You can find such an example [here](https://github.com/viperproject/gobra/blob/master/src/test/resources/regressions/features/interfaces/counterStream.gobra).

### Comparability

In Go, comparing two interfaces can cause a runtime panic. This happens if both dynamic values of an interface are incomparable. For that purpose, Gobra provides the function `isComparable`. The function takes as input an interface or a type and returns whether it is comparable according to Go. The snippet below shows a variation of a list where the values are interfaces. As an invariant, written in the `list` predicate, all elements of the list are comparable, meaning that a comparison with another interface is guaranteed to not cause a runtime panic. When the value of an implementation is put into an interface, Gobra generates the information of whether the resulting interface is comparable.


```go
package tutorial

type node struct {
  value interface{}
  next *node
}

pred list(ptr *node) {
  acc(&ptr.value) && isComparable(ptr.value) && acc(&ptr.next) && 
  (ptr.next != nil ==> list(ptr.next))
}

requires list(ptr)
pure func contains(ptr *node, value interface{}) bool {
  return unfolding list(ptr) in ptr.value == value || (ptr.next != nil && contains(ptr.next, value))
}
```

## Concurrency

Gobra supports verification of concurrent Go programs that incorporate multi-threaded programming via goroutines, channel message passing, and shared memory. Gobra ensures that every verified program is free of race-conditions. The snippet below shows code with a race condition that cannot be verified. The function `concurrentInc` spawns two goroutines that concurrently modify the value of `x`. Only one of the spawned goroutines can own the permission to `x` at any point in time. When the first goroutine is started `go inc(&x)`, the permission `acc(x)` is transferred to the goroutine. Different from a normal call, the permissions in the postcondition of the goroutine are not returned after `go inv(&x)`. As a consequence, starting the second goroutine fails, as `acc(x)` is not held anymore by the execution of `concurrentInc`.

```go
package tutorial

requires acc(x)
ensures acc(x)
func inc(x *int) {
  *x = *x + 1
}

func concurrentInc() {
  x@ := 1
  go inc(&x)
  // fails with a permission error due to a race condition
  go inc(&x)
}
```

The race condition in the previous example can be avoided by using concurrency primitives to synchronize the accesses to `x`. In Gobra, concurrency primitives make extensive use of Gobra's first-class predicates. Because of that, we first introduce predicate expressions in the next section. Afterwards, we show a race-condition free version of `concurrentInc` that uses the `Mutex` type defined in the `sync` package. Finally, we describe how to verify programs with channels.

### First-Class Predicates

Gobra has support for first-class predicates, i.e., expressions with a predicate type. First-class predicates are of type `pred(x1 T1, ..., xn Tn)`. The types `T1, ..., Tn` define that the predicate has an arity of `n` with the corresponding parameter types. 

To instantiate a first-class predicate, Gobra provides *predicate constructors*. A predicate constructor `P!<d1, ..., dn!>` partially applies a declared predicate `P` with the constructor arguments `d1, ..., dn`. A constructor argument is either an expression or a wildcard `_`, symbolizing that this argument of `P` remains unapplied. In particular, the type of `P!<d1, ..., dn!>` is `pred(u1,...,uk)`, where `u1,...,uk` are the types of the unapplied arguments. As an example, consider the declared predicate `pred sameValue(i1 int8, i2 uint32){ ... }`. The predicate constructor `sameValue!<int8(1), _!>` has type `pred(uint32)`, since the first argument is applied and the second is not. Conversely, `sameValue!<_, uint32(1)!>` has type `pred(int8)`. Finally, `sameValue!<int8(1), uint32(1)!>` and `sameValue!< _, _!>` have types `pred()` and `pred(int8, uint32)`, respectively.

The equality operator for predicate constructors is defined as a point-wise comparison, that is, `P1!<d1, ..., dn!>` is equal to `P2!<d'1, ... , d'n!>` if and only if `P1` and `P2` are the same declared predicate and if `di == d'i` for all `i` ranging from 1 to `n`.

The body of the predicate `P!<d1, ..., dn!>` is the body of `P` with the arguments applied accordingly. Like with other predicates, `P!<d1, ..., dn!>` can be instantiated and its instances may occur in assertions and in `fold` and `unfold` statements. The fold statement `fold P!<d1, ..., dk!>(e1, ..., en)` exchanges the first-class predicate instance with its body. The unfold statement does the reverse.

> **Note**: In the paper, we use the notation `P{...}` instead of `P!<...!>`. Currently, Gobra uses `!<` and `!>` as delimiters to simplify Gobra's parser. In the future, we will change to the `P{...}` syntax.

### Reasoning about Mutual Exclusion with `sync.Mutex`


In the following example, we show the function `safeInc`, a data-race free version of the previously seen function `inc` in section [Concurrency](#concurrency). The snippet uses a lock `pmutex` to synchronize the write accesses to the pointer `x`. `safeInc`'s pre and postcondition assert that `pmutex` was initialized (`acc(pmutex.LockP(), _)`) with invariant `mutexInvariant` (`pmutex.LockInv() == mutexInvariant!<x!>`), protecting the access to variable `x`. In the body of `safeInc`, first `pmutex.Lock()` is called to acquire the invariant `mutexInvariant!<x!>`. Then the first-class predicate instance is unfolded to acquire `acc(x)`, required to modify the value of `x`. At the end, the invariant is folded back again, after which `pmutex` can be released via a call to method `Unlock`.

```go
package tutorial

import "sync"

pred mutexInvariant(x *int) {
  acc(x)
}

requires acc(pmutex.LockP(), _) && pmutex.LockInv() == mutexInvariant!<x!>;
ensures  acc(pmutex.LockP(), _) && pmutex.LockInv() == mutexInvariant!<x!>;
func safeInc(pmutex *sync.Mutex, x *int) {
  pmutex.Lock()
  unfold mutexInvariant!<x!>()
  *x = *x + 1
  fold mutexInvariant!<x!>()
  pmutex.Unlock()
}
```
> **Note:** At this point in time, using semicolons is mandatory to terminate lines that end with the `!<` `!>` delimiters like in the pre and postcondition of `safeInc`. This is a known limitation of our current parser and it will be fixed when we adopt the `{` `}` delimeters for predicate constructors. 

The snippet below shows a client. First, `mutex` is allocated and the invariant is established for the first time by folding `mutexInvariant!<&x!>()`. To bind an invariant to a mutex, the `SetInv` method is called. `SetInv` expects as argument a first-class predicate with arity zero. After the invariant is bound to `&mutex`, the mutex is initialized and the permission `&mutex.LockP()` is acquired. This permission is necessary to call `Lock` and `Unlock`. At this point, the preconditions of the goroutines are satisfied and the goroutines can be started. Note that the preconditon of `safeInc` does not require write permission to the lock and thus, after starting the first goroutine, there are sufficient permissions remaining to start the second goroutine.

```go
func client() {
  x@ := 0
  mutex@ := sync.Mutex{}
  fold mutexInvariant!<&x!>()
  (&mutex).SetInv(mutexInvariant!<&x!>)
  go safeInc(&mutex, &x)
  go safeInc(&mutex, &x)
}
```

### Channels

We now shift our attention to channels. Similar to mutexes, channels must be initialized before any message can be sent and received through them. This is done via a call to the method `Init`. The method has two parameters: The first parameter is a first-class predicate with type `pred(T)`, where `T` is the type of messages communicated via the channel. It specifies properties and permissions of the data that is sent through the channel. The second parameter is a first-class predicate with type `pred()` that specifies properties and permissions that the sender obtains from the receiver when it sends a message. This is useful to model a rendez-vous of permissions in synchronous communication, which happens for unbuffered channels. In this example, and whenever the channel is buffered, the second argument will always be `PredTrue!<!>`, a predicate with arity zero whose body is `true`. Initializing a channel `c` through `Init` also produces permissions `m.SendChannel()` and `m.ReceiveChannel()`. Any read fraction of `m.SendChannel()` allows a function to send messages on `m`, whereas a read fraction of `m.ReceiveChannel()` allows a function to receive from `m`.

After initialization, the predicates passed as arguments to `Init` can be retrieved via several methods. The method `c.SendGivenPerm()` returns the invariant that must hold when the sender sends a message. The method `c.RecvGotPerm()` returns the invariant that the receiver of a message can assume. Currently, these are always the same. Likewise, `c.SendGotPerm()` and `c.RecvGivenPerm()` are the invariants that must hold when a message is received and that can be assumed when a message is sent, respectively. Currently, these are also always the same. Furthermore, they must be `PredTrue!<!>` unless the channel is unbuffered.

In the following example, the function `incChannel` receives an `*int` from channel `c` and increments the value on the received location. The first two preconditions of `incChannel` require a fraction of `c.SendChannel()` and `c.RecvChannel()` to send on and receive from channel `c`. The rest of the preconditions establish that function `incChannel` expects `c` to have been initialized with the arguments `sendInvariant!<_!>` and `PredTrue!<!>`. The predicate `sendInvariant` contains the permission to the pointer sent over the channel `c` and restricts that `*v` is positive.

The body of `incChannel` starts by folding `PredTrue!<!>()` to be able to receive from the channel. Recall that an instance of `PredTrue` must be sent from receiver to sender. It then receives from `c` and in case of success, unfolds `sendInvariant!<_!>(res)` to acquire write permissions to `res`. It then modifies `*res` and folds back `sendInvariant!<_!>(res)` to send the reply.

```go
package tutorial

pred sendInvariant(v *int) {
  acc(v) && *v > 0
}

requires acc(c.SendChannel(), 1/2)
requires acc(c.RecvChannel(), 1/2)
requires c.SendGivenPerm() == sendInvariant!<_!>;
requires c.SendGotPerm() == PredTrue!<!>;
requires c.RecvGivenPerm() == PredTrue!<!>;
requires c.RecvGotPerm() == sendInvariant!<_!>;
func incChannel(c chan *int) {
  fold PredTrue!<!>()
  res, ok := <- c
  if (ok) {
    unfold sendInvariant!<_!>(res)
    // we now have write access after unfolding the invariant: 
    *res = *res + 1
    // fold the invariant and send pointer and permission back:
    fold sendInvariant!<_!>(res)
    c <- res
  }
}
```

The next snippet shows a client. In the body, a channel `c` is created and initialized with `sendInvariant` and `PredTrue`. It then spawns a goroutine, executing `incChannel` with the initialized channel. Next, it folds `sendInvariant!<_!>(p)` to send `p` over the channel `c`. The subsequent fold of `PredTrue!<!>()` is required to satisfy the precondition of receive. At this point, `PredTrue!<!>()` must be sent from `clientChannel` (the receiver) to `incChannel` (the sender). After successfully receiving the reply, the receiver acquires `sendInvariant!<_!>(res)`, which can then be unfolded to obtain the permission to `res` and to check that its value is positive.

```go
func clientChannel() {
  var c@ = make(chan *int)

  var x@ int = 42
  var p *int = &x
  c.Init(sendInvariant!<_!>, PredTrue!<!>)
  go incChannel(c)
  assert *p == 42
  fold sendInvariant!<_!>(p)
  c <- p

  fold PredTrue!<!>()
  res, ok := <- c
  if (ok) {
    unfold sendInvariant!<_!>(res)
    assert *res > 0
    // we have regained write access:
    *res = 1
  }
}
```


## Running Gobra

A description of how to install and configure Gobra can be found in Gobra's [README](https://github.com/viperproject/gobra/blob/master/README.md) file. After setting up Gobra and obtaining a `gobra.jar` file, one can run it using the command
```bash
java -Xss128m -jar gobra.jar -i [FILES_TO_VERIFY]
```
To check the full list of flags available in Gobra, run the command
```bash
java -jar gobra.jar -h
```
