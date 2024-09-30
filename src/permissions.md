# Permissions and Heap-Manipulation

Gobra uses *implicit dynamic frames*, a variant of *separation logic*, to support reasoning about mutable heap data. In this model, each heap location is associated with an *access permission*. Access permissions are held by method executions and transferred between methods upon call and return. A method may access a location or make a call only if the appropriate permission is held. 

## Accesibility Predicate

In Go, heap locations can be thought of as addresses of pointers. The permission to access a pointer `x` is denoted by the *accessibility predicate* `acc(x)`. Having permission to a pointer implies that the pointer is not `nil`, Go's version of null. Access predicates are not duplicable resources, meaning that an accessibility predicate `P` does not entail `P && P`, where `&&` is a separating conjunction.  


The following code snippet declares a type `pair`, consisting of two integer fields, and a method `incr` that increments both fields of a `pair` pointer by an integer `n`.

```go
package tutorial

type pair struct {
  left, right int
}

// incr requires permission to modify both fields
requires acc(&x.left) && acc(&x.right)
ensures  acc(&x.left) && acc(&x.right)
ensures  x.left == old(x.left) + n
ensures  x.right == old(x.right) + n
func (x *pair) incr(n int) {
  x.left += n
  x.right += n
}
```

The method `incr` modifies heap locations by assigning to `x.left` and `x.right`. For these assignments, `incr` requires permissions to the pointers of the two fields `&x.left` and `&x.right`. It gets these permissions from its precondition `acc(&x.left) && acc(&x.right)`. When calling `incr` a caller has to transfer these permissions to the method. The postcondition `acc(&x.left) && acc(&x.right)` specifies that the permissions are returned to the caller when the call returns. As a short-hand, one can write `acc(x)` instead of `acc(&x.left) && acc(&x.right)`. Method specifications always have to specify the permissions that are returned except for pure code, which implicitly returns all permissions specified in the precondition. 


When modifying the heap it is useful to refer to the old state of the heap. An *old expression*, `old(e)` evaluates the expression `e` in the state of the precondition. As such, the postcondition `x.left == old(x.left) + n` specifies that after `incr` returns the value of `x.left` is equal to the value of `x.left` when the function was called plus `n`.

Below, we show two clients of `incr`. The function`client1` allocates a `pair` pointer. When memory is allocated, permissions to the allocated heap locations are acquired. After `p := &pair{1,2}`, permissions to both field pointers are held. The call to `incr` on the subsequent line then uses these permissions. The assertion at the end checks that `x.left == 43` holds after the call.

```go
func client1() {
  p := &pair{1,2}
  p.sumPair(42)
  assert p.left == 43
}
```

Similar to `client1`, `client2` allocates a `pair` pointer, as well. However, the allocation happens implicitly. In Gobra, when a variable is referenced (or captured by a closure), it has to be annotated with `@`. The annotation expresses that the variable is a heap location and as such involves permissions. The annotation could be inferred automatically, but we believe that inferring it makes the behavior harder to predict. Otherwise, `client1` is identical to `client2`.

```go
func client2() {
  x@ := pair{1,2} // if taking the reference of a variable should be possible, then add @
  (&x).sumPair(42)
  assert x.left == 43
}
```

Gobra offers fractional permissions, meaning that it can distinguish between different amounts of permissions. A permission amount is rational number between 0 and 1. Modifying the value of a heap location requires full permission, denoted as `acc(x, write)`. Reading from a heap location requires any non-zero permission amount. Read permissions are usually denoted as factions, e.g., `acc(x, 1/2)`. We can also write `acc(x, _)` for an arbitrary non-zero permission amount.



## Quantified Permissions

Gobra provides two key mechanisms for specifying permission to a (potentially unbounded) number of heap locations: recursive [predicates](#predicates) and *quantified permissions*.

While predicates can be a natural choice for modeling entire data  structures which are traversed in an orderly top-down fashion,  quantified permissions enable point-wise specifications, suitable for modeling heap structures that can be traversed in multiple directions, random-access data structures such as arrays, and unordered data structures such as graphs.

The basic idea is to allow resource assertions to occur within the body of a `forall` quantifier. In our example, function `addToSlice` receives a slice `s` and adds the integer `n` to each element of the slice. 

```go
package tutorial

requires forall k int :: 0 <= k && k < len(s) ==> acc(&s[k])
ensures  forall k int :: 0 <= k && k < len(s) ==> acc(&s[k])
ensures  forall k int :: 0 <= k && k < len(s) ==> s[k] == old(s[k]) + n
func addToSlice(s []int, n int) {
  invariant 0 <= i && i <= len(s)
  invariant forall k int :: 0 <= k && k < len(s) ==> acc(&s[k])
  invariant forall k int :: i <= k && k < len(s) ==> s[k] == old(s[k])
  invariant forall k int :: 0 <= k && k < i ==> s[k] == old(s[k]) + n
  for i := 0; i < len(s); i += 1 {
    s[i] = s[i] + n
  }
}
```

To be able to change the contents of the slice, the function must have access to all elements of `s`. This is provided by the precondition `forall k int :: 0 <= k && k < len(s) ==> acc(&s[k])` asserting access to all locations `&s[k]`, where `k` is a valid index of the slice. 

The function below is a client of the `addToSlice` function. The `make` call allocates the permissions required for the call.

```go
func addToSliceClient() {
  s := make([]int, 10)
  assert forall i int :: 0 <= i && i < 10 ==> s[i] == 0
  addToSlice(s, 10)
  assert forall i int :: 0 <= i && i < 10 ==> s[i] == 10
}
```
