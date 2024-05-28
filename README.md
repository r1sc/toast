# Toast language reference

Toast is a garbage collected, statically typed language that borrows heavily from the syntax style of Lua and the type system from TypeScript, ML and to an extent, C. It compiles to native code via LLVM and provides full source code debugging.

Toast uses a Mark-and-Sweep garbage collector.

Here are some design goals:
* It should be trivially intuitive to interface with C
* You shouldn't have to think about managing memory
* It should be very lightweight with few constructs
* It must provide full source code step-through debugging compatible with VSCode and/or Visual Studio
* The runtime should be lightweight and kept to a minimum
* There should be no "forced"-use libraries
* It should replace my day-to-day use of TypeScript for lab programming

## FFI

Let's start with how to interface with the C ABI, because this is always my first question when reading about a new hobby language:

```
extern function waveOutOpen(lphwo: ref uint, uDeviceID: uint, lpFormat: ref WAVEFORMATEX, dwCallback: uint, dwInstance: uint, fdwOpen: uint): uint
```

This describes the Win32 function to open an audio device. Instead of using * as a pointer marker (like in C), we use ref. Idk it just feels easier to read.

Link with `winmm.lib` from the standard Windows SDK in order to compile the above.

Let's look at `WAVEFORMATEX`, how is that defined?

```
type WAVEFORMATEX = packed(4) {
    wFormatTag: ushort,
    nChannels: ushort,
    nSamplesPerSec: uint,
    nAvgBytesPerSec: uint,
    nBlockAlign: ushort,
    wBitsPerSample: ushort,
    cbSize: ushort
}
```

To define a struct (record in Toast parlance), we just declare the fields surrounded by curly-brackets. `packed(4)` tells Toast to make sure that the field alignment is 4 bytes.

You can query the size of a record at runtime with the `sizeof WAVEFORMATEX` intrinsic. Here, it evaluates to `28`, since the field alignment is 4 bytes.

# Value and ref types

Some things are always value types, like in C:
* Integers
* Floats
* Boolean true/false

However you can put ref in front of them to pass them around as pointers.

There's no pointer arithmetic, so you don't have to explicitly deref references to access their value.

In order to write to the memory location of something declared ref, you can use the special `value at` syntax:

```
function foo(bar: ref int)
    value at bar = 8
end
```

Some elements in Toast are always `ref`, just like in C:
* Strings
* Function references
* Arrays

(TODO: Maybe this is not true...)
Unlike C, some are always passed as refs:
* Records

Records are always transferred by ref to functions (TODO: why?), but when declaring records it's necessary to make a distinction:

```
type Person = {
    name: string,
    age: uint,
    sibling: ref Person | null
}
```
Equivalent C:
```
struct Person {
    char* name;
    unsigned int age;
    Person* sibling;
};
```

Toast can do type algebra, like in TypeScript, but much more limited.
In order to use null in Toast, it needs to be part of the type.

Sometimes we want to nest records:

```
type Person = {
    name: string,
    car: {
        numWheels: uint
    }
}
```
Equivalent C:
```
struct Person {
    char* name;
    unsigned int age;
    struct {
        unsigned int numWheels;
    } car;
};
```

## Type aliases

You've seen me write `type X = ...` all over above. That's not strictly necessary, types are just named aliases for whatever is on the right side of the type. The compiler just replaces all instances of X with the right hand type at compile time. The following are equivalent:

```
type MyType = { foo: uint, bar: float }
function foo(bar: MyType)
    ..
end

function foo(bar: { foo: uint, bar: float})
    ...
end
```

## Discriminating types

If we want to accept different shapes of data to a function, we can use a union (we kind of saw that above with `Person | null`):

```
-- This is of limited use, since we cannot determine if bar is a uint or a float
function foo(bar: uint | float)
    -- ?
end
```

Just like in C, there's no runtime type information in Toast. So you cannot query the type of a value at runtime. 
However, we can use a discriminator to tell which one it is:

```
function foo(bar: { kind: 0, value: uint } | { kind: 1, value: float })
    if bar.kind == 0 then
        -- value is uint here
    else
        -- value is float here
    end
end

foo({ kind: 0, value: 123 })
foo({ kind: 1, value: 55.3 })
```

Note that there is no runtime support at all here - the compiler verifies at compile time that things are in order, but we have to provide runtime type assertions manually, like we do above. `kind` is part of the record passed info `foo`, there's no special magic.

To be clear, here's what the compiler would generate if this was C:

```
void foo(struct { unsigned char kind; union { unsigned int p1; float p2; }; } bar) {
    ...
}
```

Alhough in Toast, it doesn't matter that the discrimiated data uses the same name in both cases (`value`).

## Symbols

Using integers as discriminators is reasonable since they take up little space. However it's pretty clunky to read that code. Which one is what?
To alleviate this, Toast has @symbols. Symbols are replaced by a unique integer at compile time (they start at 1 and increment for each unique symbol):

```
function foo(bar: { kind: @int, value: uint } | { kind: @float, value: float })
    if bar.kind == @int then
        -- value is uint here
    else
        -- value is float here
    end
end

foo({ kind: @int, value: 123 })
foo({ kind: @float, value: 55.3 })
```

It's still possible to call foo with { kind: 1, value: 123 } if that's what the symbol evaluates to. However the compiler will assign this at compile time in the order the symbol is encountered, so the actual values may change depending on how they are used.

Symbols are essentially a compiler generated `#define` if you will.

Since symbols start at 1, they will always be "truthy", so you can't use them to #define DEBUG for instance.

> TODO: Maybe it should be possible to assign values to @symbols. But what happens if we assign the same value to two symbols? The compiler won't care but it could be confusing for the programmer.. Maybe it's no issue. I'll have to come back to this.

## A word on refs and values (again)

In C, we have explicit operators to access fields in value and pointer structs, `.` and `->`. Toast doesn't have that. You always do `.` and it does the dereference for you if what you are accessing is a `ref`.

## Variables

~~There are two ways to define variables in Toast:~~
* ~~const foo = "bar"~~
* let foo = "bar"

> TODO: For now, let's just do let and we can introduce const later

Let allows you to replace the value in the variable, ~~const prevents you from doing that.~~

The compiler will also hoist variables to the heap if they are captured in closures. More on this later.

## Arrays

We can define sized arrays in Toast by allocating data on the heap, just like C. However, Toast is garbage collected so we don't have to explicitly free it when we're done with it.

The formula is `alloc <number of elements expression> of <type>`.

The amount of allocated memory visible to the program = *number of elements* * *size of the type*

```
let numElements = 5
let buffer = alloc numElements of { a: int }
buffer[0] = { a: 7 }
-- or
buffer[0].a = 7
```

Arrays are not initialized, so they're probably full of junk.

You can also initialize arrays like this:
```
let shoppingList = ["eggs", "ham", "bread"]
let values = [5, 8, 3]
```

The type of each element doesn't have to be the same, the compiler uses the size of the largest element to define the array element size. But remember that there's no RTTI so make sure to discriminate the values if you need to make out which one is which.

Not useful, but legal:
```
let stuff = ["eggs", 5.2, { numWheels: 8 }]
```

You probably already figured it out, but the type of the above array is inferred to be:
```
type StuffList = [string | float | { numWheels: int }]
```

## Strings

Toast strings are wrapped in `"` as in C, but continue until a matching `"` is found, so they can wrap multiple lines. The linkebreaks are part of the string:

```
let shakespeare = 
"To be,
or \"not to be\",
that's the question?"
```

Toast strings are zero-terminated char arrays as in C.
String literals, like above are immutable, since they are stored in the read-only data section of the executable.

You can concatenate strings with `+`. Concatenating two strings result in one allocation. If you concatenate more strings, like the example below, more allocations may happen.

> TODO: For now just allocate pairwise, this can be optimized later.

```
function greet(name: string)
    return "Hello " + name + "!"
end
```

## Closures

Toast has closures:

```
function makeThing(name: string, age: uint)
    return {
        greet: function()
            return "Hello " + name + "!"
        end,
        isOlder: function(otherAge: uint)
            return age > otherAge
        end
    }
end

let thing = makeThing("foo")
print(thing.greet())
-- Hello foo!
```

This function will capture name and prevent if from being garbage collected until `thing` dies.

Only variables that is used in a closure are captured. The greet function doesn't capture age and isOlder does not capture name.

Continue reading to find out when special allocation-rules happen in closures.

## Garbage collection

Toast is garbage collected. It can allocate three things on the heap - records, arrays and scope variables. Whenever you define a record, it's on the heap. Same with arrays. Arrays are explicitly allocated with `alloc 5 of { a: int, b: string }`-style calls. Records are allocated whenever we define them, for instance:

```
let foo = {
    name: "Toast",
    age: uint(12)
}
```

The compiler won't infer field types - you must explicitly define them either on the left hand side or on the right hand side.
This is equivalent to the above:

```
let foo: { name: string, age: uint } = {
    name: "Toast",
    age: 12
}
```

Whenever you allocate something, the GC will check if it's straddled some threshold in either number of objects allocated or amount of data allocated and do a mark-sweep pass.

In performance critical code, you can alleviate this by defining and allocating *before* the critical section runs.

Consider:
```
let result = alloc 10 of { a: byte, b: byte }
for i=1, 10 do
result[i].a = i
result[i].b = i+1
end
```

In this example, allocation will only appear at the `alloc` call, the rest of the code will never trigger an allocation or a GC.

When you define a closure, any captured `let`-variables that appear on the left hand side of an assignment in any inner closure gets automatically hoisted to the heap by the GC. This way, inner closures may operate on outer data.