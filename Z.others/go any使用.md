Yes — `any` can hold **literally any Go type**, including pointer types themselves. There's no exclusion. Let me clarify precisely what `any` means.

### `any` = `interface{}` = zero-method interface

Since it requires **no methods**, every single type in Go satisfies it. That includes:

```go
var x any

x = 42            // int
x = 3.14          // float64
x = "hello"       // string
x = true          // bool
x = []int{1,2,3}  // slice
x = map[string]int{}  // map
x = MyStruct{}    // any struct
x = &MyStruct{}   // pointer to struct  ✅ pointers work too!
x = nil           // nil is valid
x = func() {}     // even functions
x = make(chan int) // even channels
```

So **pointer types are not excluded** — `*int`, `*string`, `*MyStruct`, etc. are all valid values to store in an `any`. In fact, in your original code:

```go
valuePtr := []*any{&value[0], &value[1], &value[2]}
```

`valuePtr[i]` itself is of type `*any` — a pointer, stored in a slice. That's perfectly normal; `any` doesn't care whether what's *stored inside it* is a pointer or not — and separately, you can also have a variable *of type* `*any` (pointer-to-any), which is what's happening here.

### Two different things — don't mix them up

1. **`any` holding a pointer value**:

   ```go
   var x any = &someInt   // x holds a *int
   ```

2. **A pointer to an `any` variable** (what your code does):

   ```go
   var y any = 5
   var p *any = &y        // p points at the memory location of y
   ```

Both are completely valid and unrelated to each other — Go doesn't restrict `any` based on whether the underlying type is a pointer, slice, primitive, struct, interface, or anything else.

### The only real restriction

The **only** rule is: since `any` has zero required methods, everything automatically implements it. There is no Go type — pointer or otherwise — that *cannot* be assigned to an `any` variable.