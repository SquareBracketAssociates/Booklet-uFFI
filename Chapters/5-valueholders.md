# Pointer arithmetic and value holders

One of the reasons for the power of C is that, despite the appearance of having a type system, it effectively lacks one: every value is ultimately reduced to a memory position.

An integer value, for example `x = 42`, is nothing more than a sequence of bytes stored somewhere in memory. The exact number of bytes depends on the platform and the C type, but in memory it will look something like this:  
`x = | 2A | 00 | 00 | 00 |`  
(depending on the encoding, but that is another story).

What matters here is that `x` exists at a **position in memory**, inside a chunk of memory assigned by the program, and the C compiler is able to manipulate it by performing operations on that memory location.

C exposes this fact explicitly through a fundamental operator: `&`.  
The `&` operator gives you the **address of** a variable --- that is, the position in memory where the variable is stored. Instead of giving you the value stored at that position, it gives you the position itself.

This is fundamental in C for constructing arrays and other data structures, **but it is also fundamental for passing data between functions**. If I can obtain the address of a variable, I can pass that address to another function, allowing it to modify the original variable without having direct access to it.

For example:

```language=c
void store_value(int *value) {
    *value = 42; 
}

void main() {
    int x;
    store_value(&x);
    printf("x = %d\n", x); 
    /* It will print "x = 42" */ }
```

Here, `store_value` receives not the value of `x`, but its address. By dereferencing that address, the function modifies the memory where `x` is stored.

## What does this mean in uFFI?

Historically, this has been handled in uFFI by passing a `ByteArray` to the C function. A `ByteArray` is essentially a reification of a chunk of memory, which allows C to operate on it directly.

Using the low-level mechanisms provided by uFFI, this looks like:

```language=smalltalk
| buffer x | 

buffer := ByteArray new: 4. 
self store_value: buffer. 
"store_value defined as: <ffiCall: #(void store_value(int *x))>" 
x := buffer signedLongAt: 1.
```

In short, we pass a `ByteArray` to a C function that expects a pointer to an integer, then extract the value from that memory location using a primitive.

**This is straightforward, but it is low-level and error-prone.**  
It requires detailed knowledge of memory layout, sizes, and access primitives.

## ... enter value holders

To simplify this complexity and make code easier to write and understand, we introduce **value holders**.

Value holders are not magic. They are simply a more expressive, *Pharoish* way of representing the same low-level mechanism: "a place in memory where C will store a value".

Using value holders, the same example becomes:

```language=smalltalk
| xHolder x | 
xHolder := FFIInt32 newValueHolder. 
self store_value: xHolder. 
x := xHolder value.
```
This is still more verbose than the equivalent C code, but the *meaning* of what is happening is much clearer. Let's break it down.

##### 1. Value holder creation

```language=smalltalk
xHolder := FFIInt32 newValueHolder.
```

Here we create a container --- a place where an `int32` value will be stored by C.

Yes, this requires knowing the C type and its corresponding uFFI type. But if you are calling a C function, we assume you know what you are doing 😜.

##### 2. Call the C function

```language=smalltalk
self store_value: xHolder.
```

The C function is called exactly as before. The only difference is that we pass a value holder instead of a raw memory buffer. No changes to the function declaration are required.

##### 3. Retrieve the value

```language=smalltalk
x := xHolder value.
```

You retrieve the value by simply asking the holder for it. There is no need to remember how to read an `int32` from a pointer or a byte array using low-level primitives.

This mechanism works with all basic C types defined in uFFI, including:

`FFIBool`, `FFIExternalString`, `FFIOop`, `FFIBoolean32`, `FFIFloat128`, `FFIFloat16`, `FFIFloat32`, `FFIFloat64`, `FFISizeT`,  
`FFIUInt8`, `FFIUInt16`, `FFIUInt32`, `FFIUInt64`,  
`FFIInt8`, `FFIInt16`, `FFIInt32`, `FFIInt64`, `FFILong`, `FFIULong`.

## What happens with structures, unions, and external objects?

Value holders work naturally for basic C types, but what about more complex ones?

### Structures (and unions)

When you pass a structure *by value* in C, it is always copied. This means the function receives a copy of the structure's contents and can only read them.

For example:

```language=c
typedef struct MyStruct {
    int value1;
    int value2; 
} mystructtype;

int sum(mystructtype t) { 
    return t.value1 + t.value2; 
}
```

```language=smalltalk
v := MyStruct new. 
v 
    value1: 10; 
    value2: 10. 
result := aFFILibrary sum: v.
```

This works correctly.

However, if you need the C function to **modify** the structure and observe those changes later, you must pass a *reference* --- that is, the address of the structure.

For example:

```language=c
typedef struct MyStruct {
    int value1;
    int value2; 
} mystructtype;  

void fill_values(mystructtype *t) {
    t->value1 = 10;
    t->value2 = 10; 
}
```

This example is intentionally simplified and not realistic, but it captures the essence: modifying a structure through a pointer.

### Passing structure value holders

Using value holders, this is straightforward:

```language=smalltalk
structHolder := MyStruct newValueHolder. 
aFFILibrary fill_values: structHolder. 
v := structHolder value.  
Transcript show: ('{1} + {2} = {3}' format: {
    v value1.
    v value2.
    v value1 + v value2 })
```

This ensures uniform access to structures and unions, just like any other type.

### Passing a reference to a structure

Sometimes you already have a structure instance and need to pass it by reference. This is common when a structure must be initialized first and then modified by a C function.

For this case, structures and unions provide the `referenceTo` message:

```language=smalltalk
v := MyStruct new. 
aFFILibrary fill_values: v referenceTo.
Transcript show: ('{1} + {2} = {3}' format: {
    v value1.
    v value2.
    v value1 + v value2 })
```

**Note:**  
Before this mechanism existed, uFFI relied on mangling magic for single indirection (pointer depth = 1). Because structures are internally stored in byte arrays, passing the structure itself also worked as a reference.

This behavior is subtle and relies on internal implementation details. Now that an explicit mechanism exists, **we do not recommend relying on that behavior**.
## Multiple pointer indirection (pointer depth \> 1)

In C, it is common --- especially when dealing with lists --- to encounter arguments with more than one level of indirection.
Of this cases, we will focus on arrays, since other cases follow the same pattern.
### Passing arrays
In C, arrays are just pointer arithmetic. A function declared with `int *` or `char **` often expects a list of values.

uFFI provides an abstraction for this through the `FFIArray` class. `FFIArray` can be used both to define array types and to create instances that manage storing and retrieving data through C pointers.

#### Sending arrays as part of a callout

Consider this C function:

```language=c
int sum_list(const int *list, size_t size) {
    int sum = 0;
    for (size_t i = 0; i < size; i++) {
        sum += list[i];
    }
    return sum;
}
```

The corresponding Pharo binding:

```language=smalltalk
sumList: list size: size     
    ^ self ffiCall: #(int sum_list(const int *list, size_t size))
```

Usage:

```language=smalltalk
arrayOfIntegers := FFIArray newType: FFIInt32 size: 5. 1 to: 5 do: [ :i |     arrayOfIntegers at: i put: i factorial ].  result := self sumList: arrayOfIntegers size: 5.
```

#### Retrieving arrays

Retrieving arrays is more complex, because memory allocation is often done by the C function itself.

##### Case 1: list of structures with known size

```language=c
void collect_times(time_t *times, int samples) {
    time_t *t = (time_t *)
    malloc(sizeof(time_t) * samples);
    for (i = 0; i < samples; i++) {
        t[i] = time();
    }
    *times = *t; 
}
```

Since the size is known:

```language=smalltalk
times := FFIArray newType: TimeT size: 3. 
aFFILibrary collect_times: times samples: 3.  
time1 := times at: 1. 
time2 := times at: 2. 
time3 := times at: 3.
```

##### Case 2: list of structures with unknown size

```language=c
int collect_times(time_t **times) {
    int samples = 3;
    time_t *t = malloc(sizeof(time_t) * samples);
    for (i = 0; i < samples; i++) {
        t[i] = time();
    }
    *times = t;
    return samples; 
}
```

Here the C function allocates the memory and returns the size:

```language=smalltalk
timesHolder := FFIOop newValueHolder. 
samples := aFFILibrary collect_times: timesHolder.
times := FFIArray
    fromHandle: timesHolder value
    type: TimeT
    size: samples.
time1 := times at: 1.
time2 := times at: 2.
time3 := times at: 3.  
"NOTICE THAT IN THIS CASE IT IS YOUR RESPONSIBILITY TO RELEASE THE ALLOCATED MEMORY"
```

#### Other cases

Even though we focused on arrays, the same pattern applies to all cases involving pointer depth greater than one: you pass a `FFIOop` value holder and interpret the result accordingly.
