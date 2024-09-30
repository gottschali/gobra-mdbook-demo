
## Structure of annotated Go Programs

Like Go, annotated programs consist of a *package clause* followed by a list of *imports declarations*, followed by a list of *top-level declarations*.
In the following subsections, each of these components is described briefly.

### Package Clauses

```go
package pkgName
```

- The `package` keyword defines the package to which a file belongs.
- Every Gobra file is part of exactly one package.

### Import Declarations

```go,editable
import (
    "strings"
    "sync"
    "time"
)
```
- Importing packages works as in Go.
- For an imported Go package, Gobra must find `.gobra` files with a matching package declaration in its included directories. Gobra will use those files as header files for the package declarations. 
- Files belonging to the same package have to be located in the same directory.
- The flag `-I path1 ... path2` includes the provided paths.
<!-- Gobra provides an incomplete but growing support for the Go standard library. Currently, it has partial support for the packages `encoding/binary`, `net`, `strconv`, `strings`, `sync`, and `time`. -->

### Top-Level Declarations

#### Functions and Methods

```go
requires ... // preconditions 
ensures  ... // postconditions 
trusted      // trusted flag (optional)
func min(xs []int) int {
  ... // function body (optional)
}
```

```go,editable
requires ... // preconditions 
ensures  ... // postconditions 
trusted      // trusted flag (optional)
func (xs *list) contains(value int) bool {
  ... // method body (optional)
}
```
- Functions and methods can be annotated with pre and postconditions. If omitted, they default to `true`.
- Functions and methods are verified modularly. A call only uses the specification of the callee and cannot peek inside the body.
- The body (including `{}`) may be omitted. In this case, Gobra assumes that the specification holds.
- A function or method can be annotated with the `trusted` flag. Gobra then assumes the specification holds and ignores the body.

#### Predicates

```go
pred listPermissions(xs *list) {
  ... // an assertion parameterized in the arguments (optional)
}
```

- Not part of the Go language and declared with the keyword `pred`.
- Typically used to abstract over assertions and to specify the shape of recursive data structures.
- See the [section on predicates](#predicates) for details.

#### Type Definitions

```go
type pair struct {
  left, right int
}
```

- Type declarations work as in Go.
- In this example, `pair` is the newly declared type whose underlying type is `struct{ left, right int }`. 


#### Constants

```go
const maxScore = 60 // untyped constant
const minScore int = 10 // typed constant
```
- Constants work as in Go.
- The value of a constant must be known at compile-time.
- Since constants cannot be assigned to, permissions are not required to access them.

#### Implementation Proofs

```go
(*counter) implements stream {
  ...
}
```

- Explained in [the section on interfaces](#interfaces)
