# Basic Annotations

All examples shown in this tutorial can be found [here](../src/test/resources/regressions/examples/tutorial-examples/). We start with a simple function `sum` computing the sum of the integers from 0 to `n`. The table below shows a Go implementation (left) and a corresponding annotated version (right).

<table>
<tr>
<td> Go code </td> <td> Gobra code </td>
</tr>
<tr>
<td> 


```go
package tutorial



func sum(n int) (sum int) {
  sum = 0



  for i := 0; i <= n; i++ {
    sum += i
  }
  return sum
}
```

</td>
<td>

```go
package tutorial

requires 0 <= n // precondition
ensures  sum == n * (n+1)/2 // postcondition
func sum(n int) (sum int) {
  sum = 0

  invariant 0 <= i && i <= n + 1 
  invariant sum == i * (i-1)/2
  for i := 0; i <= n; i++ {
    sum += i
  }
  return sum
}
```

</td>
</tr>
</table>

In addition to the Go code, the annotated version has a *method specification*, in the form of a pre and postcondition, and an invariant for the loop. The precondition `0 <= n` restricts the argument `n` to non-negative values. A caller of `sum` has to guarantee this assertion. The postcondition `sum == n * (n+1)/2` expresses an assertion that holds after a call returns, which has to be guaranteed by the function body. Pre and postconditions can be split into multiple `requires` and `ensures` clauses. The invariant `0 <= i && i <= n + 1` and `sum == i * (i-1)/2` is used to the verify the loop. An invariant has to hold before a loop and must hold at the end of the loop body. Note that the initial statement `i := 0` is considered to happen before the loop. After the loop, we know that the invariant and the negated loop condition holds. At this point in time, Gobra does not prove termination so only partial correctness is provided.

As assertions, Gobra supports Go's side-effect-free, determinsitic boolean expressions (e.g. `x > y + z`), implications (`==>`), conditionals (`cond ? e1 : e2`), separating conjunctions (`&&`), universal quantifiers (e.g. `forall x int :: x >= 5 ==> x >= 0`), and existential quantifiers (e.g. `exists x int :: n > 42`). More specific assertions, e.g. for the access of heap locations and interfaces are introduced in subsequent sections.

When a function or method does not have side-effects, such as modifying heap-allocated structures, and is deterministic, it may be annotated with a `pure` clause. We refer to these functions and methods as `pure`. Different from non-pure code, calls to pure code can be used in specifications. For that reason, they are a powerful specification mechanism. Consider the following snippet. The function `isEven` is pure and returns whether the argument is even. The `halfRoundedUp` uses `isEven` in its method specifiction.

```go,editable
package tutorial

ensures res == (n % 2 == 0)
pure func isEven(n int) (res bool) {
  return n % 2 == 0
}

ensures isEven(n) ==> res == n / 2
ensures !isEven(n) ==> res == n / 2 + 1
func halfRoundedUp(n int) (res int) {
  if isEven(n) {
    res = n / 2
  } else {
    res = n / 2 + 1
  }
  return res
}
```

Currently, the `pure` modifier has to be written after the pre and postcondition and before the `func` keyword. Furthermore, pure functions and methods have strict syntax constrains, the body must be a single return statement that returns a single deterministic side-effect free expression.


