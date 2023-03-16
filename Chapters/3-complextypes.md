## Complex TypesWe have seen in the last two chapters that types are a big deal in FFI call-outs, since they govern how values are marshalled between C and Pharo.The previous chapter explored the marshalling rules of basic types such as integers, floating point numbers, booleans and characters.But in addition to these basic types, existing libraries may make use of more complex data-types, i.e., data-types that are built from simpler data-types.This chapter builds on top of the knowledge of previous chapters to introduce these more complex types.This chapter starts by showing how to define type aliases. Type aliases are user-defined alternative names for other types, usually used to improve the readability of the code.In addition, you'll see further in this chapter how uFFI exploits them to define more complex types.uFFI provides support to map C complex data-types, such as arrays, structures, enumerations and unions.Arrays are data-types defining a bounded sequence of values of a single data-type, e.g., a sequence of 10 integers.Structures are data-types defining a collection of values of heterogeneous data-types, e.g., a group of an integer and two booleans.Enumerations are data-types defining a finite set of named values, e.g., the characters between a and z.Unions are data-types defining a single value that can be interpreted with different internal representations, e.g., we may want to see a float as an int to extract its mantissa.### Defining Type AliasesA type alias is a user-defined alternative name for a type, which is useful in many different scenarios.One example is improving code readability by creating domain specific types, e.g., `age` => `unsigned int`.As another example, external libraries can come with their own custom types and aliases, so having the same aliases in our FFI bindings can simplify writing those bindings.#### A first type aliasIn uFFI, type aliases are created with class variables: the class variable name is the alias name, while its value is the aliased type.For example, we define our `age` => `unsigned int` alias as follows in our `FFITutorial` class, and then execute the initialize method to make it run.```language=smalltalkObject subclass: #FFITutorial
  instanceVariableNames: ''
  classVariableNames: 'Age'
	package: 'FFITutorial'
  
FFITutorial class >> initialize
  Age := #uint```Once our type alias is defined, and the class side `initialize` executed, we can use that type alias anywhere in our bindings in the class hierarchy below `FFITutorial`. For example, we can define our `abs()` binding as follows.```language=smalltalkFFITutorial class >> abs: n [
	^ self ffiCall: #( Age abs ( Age n ) )
]```#### Valid values for type aliasesAs we have seen above, uFFI type aliases are defined by normal assignments to class variables:```language=smalltalkAge := #uint```uFFI type alias names, on the left of the assignment, can have any name accepted as a class variable name.Although other names are technically accepted in Pharo class definitions, the convention is that class variables should be capitalized.The value of a type alias, on the right of the assignment, could be either:- a symbol with a type identifier to be resolved by uFFI, e.g., `#'int'`, as shown above or;- an already resolved type object, which we will study in subsequent sectionsBefore executing a call-out, uFFI verifies all types used in the call-out can be resolved to valid types, and will throw an exception if an error occurs while resolving them.#### Sharing type aliases with shared poolsuFFI does not enforce how bindings should be structured by a developer.A developer could choose to put all bindings in a single class, or organize them in several classes, even amongst several packages. Regardless of how the code is structured, it is usually useful to have user-defined type aliases available in all classes using the bindings. For this purpose, uFFI supports structuring type aliases in shared pools.A typical usage of shared pools to define uFFI types is to define a `MyLibraryTypes` shared pool as follows, as we did before to define constants:```language=smalltalkObject subclass: #FFITutorialTypes
  instanceVariableNames: ''
  classVariableNames: 'Age'
	package: 'FFITutorial'
  
FFITutorialTypes class >> initialize
  Age := #uint```We can then import the type aliases in the shared pool by ```language=smalltalkObject subclass: #FFITutorial
  ...
  poolDictionaries: 'FFITutorialTypes'
  ...```### ArraysArrays are a bounded sequence of contiguous values.In Pharo, an array object can contain any object, so a single array can contain objects of different types.For example, the following code snippet shows how a single array can contain integers, floats, strings and others.```language=smalltalkanArray := Array new: 7.
anArray at: 1 put: 3.1415.
anArray at: 5 put: 42.
anArray at: 7 put: 'Hello World'.

anArray.
 => #(3.1415 nil nil nil 42 nil 'Hello World')```In addition, Pharo arrays can be safely accessed without producing buffer over/underflows, because it performs bound checks on each array access. In other words, accessing outside the bounds of a Pharo array yields an exception instead of accessing data **outside** the array.```language=smalltalkanArray := Array new: 1.

anArray at: 2
 => Out of Bounds Exception!```C arrays behave somewhat like Pharo arrays: they are contiguous sequences of values.However, C arrays are constrained to contain values of a single type, and accessing outside of their bounds is not checked before access, allowing buffer over/underflows.Because of these differences, and others in their internal representation, uFFI does not automatically marshall Pharo arrays into C arrays. Instead, uFFI provides a specialized array class to manipulate arrays: the `FFIArray` class.#### Creating `FFIArray`s`FFIArray`s are created using the `newType:size:` or the  `externalNewType:size:` instance creation methods. The former will allocate an array in the Pharo heap, while the latter one will allocate the array in the C heap.```language=smalltalk"In the pharo heap"
array := FFIArray newType: #char size: 10.

"In the C heap"
array := FFIArray externalNewType: #char size: 10.````FFIArray`s allocated in the C heap are not moved and their memory is not released automatically.It is the developer's responsibility to free it.On the other hand, `FFIArray`s allocated in the Pharo heap can be moved by the garbage collector, so they should be pinned in memory before being safely used in FFI calls.Also, `FFIArray`s in the Pharo heap are managed by Pharo's garbage collector, and will be collected if no other Pharo objects reference it. Be careful, an `FFIArray` in the Pharo heap referenced from the C heap will still be garbage collected making the pointer in the C heap a dangling pointer.#### Manipulating `FFIArray` instancesThe elements in an `FFIArray` are accessed as any other Pharo array, using the `#at:` and `#at:put:` methods.Its size is accessed with the `#size` method, using 1-based indexes like in Pharo.```language=smalltalkarray at: 1 "for the first element".
array at: n "for the nth element".```If the array is allocated in the Pharo heap, array accesses will be bound checked and throw an exception in case of out-of-bounds access. Otherwise, if the array is allocated in the C heap, array accesses may run into buffer over/underflows.#### Reusable `FFIArray` typesFrom time to time, we need to create several array instances of the same type and size.Besides instantiating single arrays, `FFIArray` can define array types, using the `newArrayType:size:` method.An array type knows the types of its elements and its size and we can simply allocate it using the `new` or `externalNew` messages to allocate it in the Pharo heap or the C heap respectively.```language=smalltalkchar128Type := FFIArray newArrayType: #char size: 128.

"In Pharo heap"
newArrayOf128Chars := char128Type new.

"In C heap"
newArrayOf128Chars := char128Type externalNew.````FFIArray`s created this way can be used like any other `FFIArray`.We will see in the next section how this array definition is useful to combine arrays inside structures.### StructuresA structure is a data-type that joins together a group of variables, so-called fields.Each field of a structure has a name and a type.Structures are often used to group related values, so they can be manipulated together.For example, let's consider a structure that models a fraction, i.e., a number that has a numerator and a denominator. Both numerator and denominator can be defined as fields of type `int`.Such a fraction structure data-type, and a function calculating its double precision floating point number equivalent, are defined in C as follows:```language=ctypedef struct
{
	int numerator;
	int denominator;
} fraction;

double fraction_to_double(fraction* a_fraction){
  return a_fraction -> numerator / (double)(a_fraction -> denominator);
}```#### Defining a structure with `FFIStructure`Structures are declared in uFFI as subclasses of the `FFIStructure` class defining the same fields as defined in C.For example, defining our fraction structure is done as follows, defining a subclass of `FFIStructure`, a `fieldsDesc` class-side method returning the specification of the structure fields, and finally sending the `rebuildFieldAccessors` message to the structure class we created.```language=smalltalkFFIStructure subclass: #FractionStructure
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'FFITutorial'

FractionStructure class >> fieldsDesc [
	^ #(
		int numerator;
		int denominator;
		)
]

FractionStructure rebuildFieldAccessors.```Doing this will automatically generate some boilerplate code to manipulate the structure.You will see that the structure class gets redefined as follows, containing some auto-generated accessors.```language=smalltalkFFIStructure subclass: #FractionStructure
	instanceVariableNames: ''
	classVariableNames: 'OFFSET_DENOMINATOR OFFSET_NUMERATOR'
	package: 'FFITutorial'

FractionStructure >> denominator [
	"This method was automatically generated"
	^handle signedLongAt: OFFSET_DENOMINATOR
]

FractionStructure >> denominator: anObject [
	"This method was automatically generated"
	handle signedLongAt: OFFSET_DENOMINATOR put: anObject
]

FractionStructure >> numerator [
	"This method was automatically generated"
	^handle signedLongAt: OFFSET_NUMERATOR
]

FractionStructure >> numerator: anObject [
	"This method was automatically generated"
	handle signedLongAt: OFFSET_NUMERATOR put: anObject
]```Once a structure type is defined, we can allocate structures from it using the `new` and `externalNew` messages, that will allocate it in the Pharo heap or the external C heap respectively.```language=smalltalk"In Pharo heap"
aFraction := FractionStructure new.

"In C heap"
aFraction := FractionStructure externalNew.```We read or write in our structure using the auto-generated accessors.```language=smalltalkaFraction numerator: 40.
aFraction denominator: 7.```And we can use it as an argument in a call-out by using its type.```language=smalltalkFFITutorial >> fractionToDouble: aFraction [
  ^ self ffiCall: #(double fraction_to_double(FractionStructure* a_fraction))
]

FFITutorial new fractionToDouble: aFraction.
>>> 5.714285714285714```#### Structures by copy or by referenceStructures in C can be passed as an argument both by copy and by reference.A structure passed by copy means that a copy of the entire structure has to be made and sent to the calling function.A structure passed by reference means that a pointer to the same structure is shared between the caller and callee.In C, we can distinguish between these two in two cases: 1\) by the appearance of the pointer type modifier **\*** in type declarations, and 2\) by the need to explicitly send a pointer to a function expecting pointers.```language=cint print_by_pointer(fraction* a_fraction_reference);
int print_by_copy(fraction a_fraction_copy);

...

fraction f;

//The function below requires a copy, so just send f, the compiler takes care
print_by_copy(f);

//The function below requires a pointer, so dereference f
print_by_pointer(&f);```In uFFI, such a difference is also reflected in FFI bindings by using a pointer type \(or not\):```language=smalltalkFFITutorial >> printByPointer: aFraction [
  ^ self ffiCall: #(int print_by_pointer(FractionStructure* a_fraction))
]

FFITutorial >> printByCopy: aFraction [
  ^ self ffiCall: #(int print_by_copy(FractionStructure a_fraction))
]```However, in contrast with C, when supplying a structure object as an argument, Pharo does not require the developer to worry about dereferencing.Just supply the structure object as an argument, and uFFI will take care of dereferencing if needed.```language=smalltalk"In C heap"
aFraction := FractionStructure externalNew.

aFraction numerator: 40.
aFraction denominator: 7.

"Both of these work"
FFITutorial new printByPointer: aFraction.
FFITutorial new printByCopy: aFraction.```#### Structures embedding arraysArrays in C can appear embedded in structures, like the one defined below, which contains an array of four ints.```language=cstruct {
	int some_array[4];
}```Unlike a struct containing _a pointer_ to an array, structs created from the definition above will contain the entire array allocated within the struct. Choosing between a struct that embeds an array or one that references an array through a pointer is a responsibility of the author of the C library to which we are binding, and it is outside the scope of this booklet. However, these different structures will have to be defined differently in uFFI. Defining the structure above in uFFI requires that we define an array type of size 4 for our `some_array` field.We can define such a user-defined type as a type alias in our type pool, import our pool into our structure class and then use our type in our field definitions.```language=smalltalkFFITutorialTypes class >> initialize [
  int4array := FFIArray newArrayType: #int size: 4.
]
  
FFIStructure subclass: #EmbeddingArrayStructure
  instanceVariableNames: ''
  classVariableNames: ''
  poolDictionaries: 'FFITutorialTypes'
  package: 'FFITutorial'
  
EmbeddingArrayStructure class >> fieldsDesc [
  ^ #(
  int4array some_array;
  )
]

EmbeddingArrayStructure rebuildFieldAccessors.```#### Structure AlignmentStructures are usually organised in memory as a contiguous region containing all fields in the order they were defined.However, compilers usually align structure fields to simplify access to them, by adding some **padding**, i.e., hidden fields that occupy some space to force subsequent fields to move to the desired position.For example, consider a structure with two fields `a` and `b` of types `char` and `int` respectively.Although the `char a` field only occupies 1 byte, the second field `b` starts in the fifth byte: it is aligned to 4 bytes. This means the compiled version of such a struct adds 3 bytes of padding between our two fields.```language=c// We define
struct {
  char a;
  int b;
}

// The compiler defines
struct {
  char a;
  char padding1[3];
  int b;
}```uFFI handles padding and alignments automatically, respecting the C standard behavior.We can define the structure above using uFFI as:```language=smalltalkFFIStructure subclass: #AlignmentExampleStructure
instanceVariableNames: ''
  classVariableNames: ''
  package: 'FFITutorial'

AlignmentExampleStructure class >> fieldsDesc [
  ^ #(
  	char a;
  	int b;
  	)
]

AlignmentExampleStructure rebuildFieldAccessors.```And then test that the fields are correctly aligned: `a` is the first byte, `b` is the fifth:```language=smalltalAlignmentExampleStructure classPool at: #OFFSET_A.
>>> 1
AlignmentExampleStructure classPool at: #OFFSET_B.
>>> 5```#### Packed StructuresFrom time to time we will find libraries that use **packed structures**.Packed structures are structures that are compiled without some or all of their padding.For example, some compilers will use the pragma `pack` to tweak the alignment of structures.```language=c#pragma pack(push)  /* push current alignment to stack */
#pragma pack(1)     /* set alignment to 1 byte boundary */

struct {
  char a;
  int b;
}

#pragma pack(pop)   /* restore original alignment from stack */```Some other compilers will have directives to specify packing at the level of a field:```language=cstruct {
  char a;
  int b __attribute__((packed));
}```uFFI provides support for mapping **packed structures** through the `FFIPackedStructure` class, which is a subclass of `FFIStructure` that redefines how fields are aligned. `FFIPackedStructure` avoids all padding, creating a single packed structure where each field follows the next one. Consider the example of `FFIPackedStructure` below, mapping our packed structure above.```language=smalltalkFFIPackedStructure subclass: #PackedAlignmentExampleStructure
instanceVariableNames: ''
  classVariableNames: ''
  package: 'FFITutorial'

PackedAlignmentExampleStructure class >> fieldsDesc [
  ^ #(
  	char a;
  	int b;
  	)
]

PackedAlignmentExampleStructure rebuildFieldAccessors.```Differently from the non-packed structure of a couple of sections ago, this packed structure shows that both fields are contiguous: `a` is the first byte, `b` is the second:```language=smalltalkPackedAlignmentExampleStructure classPool at: #OFFSET_A.
>>> 1
PackedAlignmentExampleStructure classPool at: #OFFSET_B.
>>> 2```### EnumerationsEnumerations are data-types defining a finite set of named values.For example, let's say we want to create a data-type to identify the different positions of players in a football match: goalkeeper, defender, midfielder, forward. Such a data-type can be defined in C as an enumeration as follows:```language=ctypedef enum {
  goalkeeper,
  defender,
  midfielder,
  forward
} position;```We can then use `position` as a type, and the values defined within it as valid values for `position`.```language=cposition myPosition = defender;```#### The values of C enumerationsTo better understand how to map C enumerations using uFFI, we must before understand how C assigns values to each of the elements in the enumeration.Internally, C assigns to each of the elements of the enumeration a sequencial numeric value starting from 0 \(zero\).In other words, `goalkeeper` has a value of 0, `defender` has a value of 1, and so on.C allows developers to specify the values they want too, using an assignment-like syntax.```language=ctypedef enum {
  goalkeeper = 42,
  defender,
  midfielder,
  forward
} position;```We can explicitly assign values to any of the elements of the enumeration.We may leave values without explicit values, in which case they will be automatically assigned the value following the previous value. And finally, many elements in the enumeration may have the same value.The example enumeration below shows these subtleties.```language=c#include <assert.h>
#include <limits.h>

enum example {
    example0,            /* will have value 0 */
    example1,            /* will have value 1 */
    example2 = 3,        /* will have value 3 */
    example3 = 3,        /* will have value 3 */
    example4,            /* will have value 4 */
    example5 = INT_MAX,  /* will have value INT_MAX */
    /* Defining a new value after this one will cause an overflow error */
};```#### Defining an enumeration using `FFIEnumeration`Enumerations are declared in uFFI as subclasses of the `FFIEnumeration` class, which similarly define their elements and values.Note that in uFFI, a value must be explicitly provided for each element.For example, defining our example enumeration is done as follows, defining a subclass of `FFIEnumeration`, an `enumDecl` class-side method returning the specification of the enumeration elements, and finally sending the `initialize` message to the enumeration class we created.```language=smalltalkFFIEnumeration subclass: #ExampleEnumeration
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'FFITutorial'

ExampleEnumeration class >> enumDecl [
	^ #(
    example0 0
    example1 1
    example2 3
    example3 3
    example4 4
    example5 2147483647
		)
]

ExampleEnumeration initialize.```Doing this will automatically generate some boilerplate code to manipulate the enumeration.You will see that the enumeration class gets redefined as follows, creating and initializing a class variable for each of its elements.```language=smalltalkFFIEnumeration subclass: #ExampleEnumeration
  instanceVariableNames: ''
  classVariableNames: 'example0 example1 example2 example3 example4 example5'
  package: 'FFITutorial'```To use the values of enumerations in our code, it is enough to import it as a pool dictionary, since uFFI enumerations are shared pools.```language=smalltalkObject subclass: #FFITutorial
  ...
  poolDictionaries: 'ExampleEnumeration'
  ...```### UnionsUnions are data-types defining a single value that can be interpreted with different internal representations.For example, the next piece of C code defines a type `float_or_int` that can be seen as a float or as an int.```language=ctypedef union {
  float as_float;
  int as_int;
} float_or_int;

float_or_int number;
number.as_float = 3.14f;

printf("Integer representation of PI: %d\n", number.as_int);```producing the next output:```Integer representation of PI: 1078523331```#### Defining a union using `FFIUnion`Similar to the way structures are handled, unions are declared in uFFI as subclasses of the `FFIUnion` class, defining the same fields as defined in C.For example, defining our `float_or_int` union is done as follows, defining a subclass of `FFIUnion`, a `fieldsDesc` class-side method returning the specification of the union's fields, and finally sending the `rebuildFieldAccessors` message to the union class we created.```language=smalltalkFFIUnion subclass: #FloatOrIntUnion
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'FFITutorial'

FloatOrIntUnion class >> fieldsDesc [
	^ #(
		float as_float;
		int as_int;
		)
]

FloatOrIntUnion rebuildFieldAccessors.```Doing this will automatically generate some boilerplate code to manipulate the values inside the union.You will see that the union class gets redefined like structures did, containing some auto-generated accessors.```language=smalltalkFFIStructure subclass: #FloatOrIntUnion
	instanceVariableNames: ''
	classVariableNames: 'OFFSET_DENOMINATOR OFFSET_NUMERATOR'
	package: 'FFITutorial'

FloatOrIntUnion >> as_float [
	"This method was automatically generated"
	^handle floatAt: 1
]

FloatOrIntUnion >> as_float: anObject [
	"This method was automatically generated"
	handle floatAt: 1 put: anObject
]

FloatOrIntUnion >> as_int [
	"This method was automatically generated"
	^handle signedLongAt: 1
]

FloatOrIntUnion >> as_int: anObject [
	"This method was automatically generated"
	handle signedLongAt: 1 put: anObject
]```#### Using the defined union typeOnce a union type is defined, we can create a union using the `new` and `externalNew` messages, which will allocate it in the Pharo heap or the external C heap respectively.```language=smalltalk"In Pharo heap"
aFloatOrInt := FloatOrIntUnion new.

"In C heap"
aFloatOrInt := FloatOrIntUnion externalNew.```We read or write in our union using the auto-generated accessors.```language=smalltalkfoi as_float: 3.14.
foi as_int.
>>> 1078523331```And we can use it as an argument in a call-out by using its type.```language=smalltalkFFITutorial >> firstByte: float_or_union [
  ^ self ffiCall: #(char float_or_int_first_byte(FloatOrIntUnion* float_or_union))
]```### ConclusionIn this chapter we have seen how complex C data-types can be mapped with uFFI.In contrast with basic types, which are automatically marshalled between Pharo and C, uFFI does not automatically marshall complex data-types.The reasoning behind this decision is that the memory layout of C complex data-types is entirely different than Pharo objects.Instead of automatically marshalling Pharo objects into these complex data-types, uFFI reifies them and allows developers to manipulate them through messages in normal Pharo code.uFFI provides representations for arrays, structures, enumerations and unions.Moreover, these types can be combined, through the usage of type aliases.In the next chapter, we will study how we can define FFI bindings in an object-oriented fashion, using encapsulation, inheritance and delegation.