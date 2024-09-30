# Debug

## Go block
```go,editable
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

## Gobra Block
```gobra,editable
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
## Highlight JS
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
