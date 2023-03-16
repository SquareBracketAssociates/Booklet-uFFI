## Designing with uFFI and FFI ObjectsIn the previous chapters, we have studied the uFFI mechanics necessary to bind an external C library.We have studied in particular how to call functions, how basic types are marshalled, and how we can manipulate more complex types.However, we have not yet discussed how those call-out bindings and types should be structured in a project.This chapter presents several ways to organize library bindings.The first approach presented is the naïve-yet-simple _single library object_ that organises the bindings as a façade.In addition, uFFI allows one to take a second approach exploiting the object-oriented nature of Pharo, in particular splitting the binding into different objects and encapsulating the data they manipulate.For this, we exploit the concepts of **external objects** and **opaque objects**.### First approach: single library façade objectThe first approach for organizing external library bindings is to follow the Façade design pattern.In other words, there is a single object that concentrates all bindings and types of our library.This approach is useful to see and understand the bindings as a whole, since they are all together.#### The `FFILibrary` as a single point of accessuFFI proposes an ideal place to apply this pattern: our library object \(which is a subclass of `FFILibrary`\).We have seen in previous chapters that when defining a call-out, we need to specify the external library where the function is located, either in the definition of the call-out using the `ffiCall:library:` method:```language=smalltalkFFITutorial >> abs: n [
  self ffiCall: #( int abs (int n) ) library: MyLibC
]```Or by redefining the method `ffiLibrary` to specify the library for all call-outs in the class, thereby avoiding the need to do so in every call-out.```language=smalltalkFFITutorial >> abs: n [
  self ffiCall: #( int abs (int n) )
]

FFITutorial >> ffiLibrary [
  ^ MyLibC
]```When we choose to use the Façade pattern for our bindings, uFFI provides a convenience behaviour: for all subclasses of `FFILibrary`, the method `ffiLibrary` returns `self`:```language=smalltalkFFILibrary >> ffiLibrary [
  ^ self
]```This convenience behaviour allows binding developers to place call-out bindings inside the library object, and avoid declaring the `ffiLibrary` at all.By using this pattern, the library object is not only the object that knows how to find the external library and find its functions: it becomes also a reification of the external library, including its functions. #### Redefining `MyLibC` as a façadeFollowing the pattern explained above, we can now turn our `MyLibC` example library from the first chapter into a Façade. We will move the call-out methods, `ticksSinceStart` and `time`, from the client class, `FFITutorial`, to our library class and eliminate the need to explicitly specify the library at all in the call-outs:```language=smalltalkFFILibrary subclass: #MyLibC
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'UnifiedFFI-Libraries'

MyLibC >> unixLibraryName [
	^ 'libc.so.6'
]

MyLibC >> macLibraryName [
	^ 'libc.dylib'
]

MyLibC >> win32LibraryName [
	"While this is not a proper 'libc', MSVCRT has the functions we need here."
	^ 'msvcrt.dll'
]

MyLibC >> ticksSinceStart [
	^ self ffiCall: #( uint clock() )
]

MyLibC >> time [
	^ self ffiCall: #( uint time( NULL ) )
]```Which can be used as any normal object now:```language=smalltalkMyLibC new time```We leave it as an exercise for the reader to explore the differences between putting the call-out bindings as instance-side or class-side methods.As the reader might assume, this approach is the strongest when we are binding a small library or a small subset of a large library.The main advantage is that the entire binding can be understood by taking a look at a single class.However, as soon as bindings grow in complexity, we need to re-structure and refactor our bindings into different objects, as we will see in the following sections.### Extracting behaviour into objectsThe FFI library façade object risks becoming an unwieldy god object when the C library is big or complex.To cope with this inherent complexity, uFFI provides several mechanisms to extract FFI call-outs into objects.**External objects** have special marshalling rules which, combined with `self` arguments, allow us to design nice object-oriented APIs.Also, uFFI provides support for **Opaque objects**, which are external objects whose internal implementation is not exposed by the FFI library.#### External object marshalling rulesExternal objects are uFFI objects representing objects from the external library.We have already seen several external objects such as structures, unions and enumerations, which require creating subclasses of `FFIStructure`, `FFIUnion` or `FFIEnumeration` respectively.uFFI external objects are normal Pharo objects wrapping an external address, held in a _handle_ instance variable.This _handle_ instance variable is defined in `ExternalObject`, the common superclass of all external objects.```language=smalltalkObject subclass: #ExternalObject
  instanceVariableNames: 'handle'
  classVariableNames: ''
  package: 'FFI-Kernel'```In the previous examples, we have also seen that we can use external objects in our call-outs by specifying their class name.Using external objects in call-outs is possible because uFFI defines special marshalling rules for such objects.- When a uFFI external object is sent as an argument, uFFI will use its _handle_ in the call-out, thus passing the actual memory pointer.- When a uFFI external object is expected as a return value, uFFI expects that the returned value is a pointer and it instantiates an external object of the specified type, setting that pointer as its _handle_.#### An example of external object marshallingTo study how marshalling of external objects works, we will re-introduce the fraction example from the last chapter.Consider the fraction structure, both in C:```language=ctypedef struct
{
	int numerator;
	int denominator;
} fraction;

double fraction_to_double(fraction* a_fraction){
  return a_fraction -> numerator / (double)(a_fraction -> denominator);
}```And its uFFI binding in Pharo:```language=smalltalkFFIStructure subclass: #FractionStructure
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'FFITutorial'

FractionStructure class >> fieldsDesc [
	^ #(
		int numerator;
		int denominator;
		)
]```#### Marshalling external object argumentsAs an example, let's consider `fraction_to_double` above and its usage below:```language=smalltalkFFITutorial >> fractionToDouble: aFraction [
  ^ self ffiCall: #(double fraction_to_double(FractionStructure* a_fraction))
]

FractionStructure rebuildFieldAccessors.
aFraction := FractionStructure externalNew.
aFraction numerator: 40.
aFraction denominator: 7.
FFITutorial new fractionToDouble: aFraction.
>>> 5.714285714285714```In the example above, we see that the call-out binding specifies a pointer to our Pharo `FractionStructure` type.However, when using this binding, we simply pass a normal Pharo object and uFFI handles the rest.Using external object types as function arguments is actually uFFI syntax sugar for the following \(not so nice\) binding.This binding makes use of pointers with `void*` argument types and breaks the encapsulation of our structure to access its internal `handle`, coupling itself with the internals of uFFI.```language=smalltalkFFITutorial >> fractionToDouble: aFraction [
  ^ self ffiCall: #(double fraction_to_double(void* a_fraction))
]

...
FFITutorial new fractionToDouble: aFraction handle.
>>> 5.714285714285714```#### Marshalling external object return valuesExternal object types can be used as function return values, too.Consider the following C function, which creates and returns a fraction struct, and its uFFI binding.Using `FractionStructure` as the return type of our binding, we tell uFFI to take the pointer returned from the function and create a `FractionStructure` with that pointer as its handle.```language=cfraction* make_fraction(int numerator, int denominator){
  fraction* f = (fraction*)malloc(sizeof(fraction));
  f -> numerator = numerator;
  f -> denominator = denominator;
  return f;
}``````language=smalltalkFFITutorial >> newFractionWithNumerator: numerator denominator: denominator [
  ^ self ffiCall: #(FractionStructure* make_fraction(int numerator, int denominator))
]

aFraction := FFITutorial new newFractionWithNumerator: 40 denominator: 7.
FFITutorial new fractionToDouble: aFraction.
>>> 5.714285714285714```As with arguments, external object return types are actually uFFI syntax sugar for the following \(again not so nice\) binding.This binding makes use of a pointer with `void*` return type and manually initializes a structure from it, coupling itself with the internals of uFFI.```language=smalltalkFFITutorial >> newFractionWithNumerator: numerator denominator: denominator [
  ^ self ffiCall: #(void* make_fraction(int numerator, int denominator))
]

aFraction := FractionStructure fromHandle: (FFITutorial new newFractionWithNumerator: 40 denominator: 7).
FFITutorial new fractionToDouble: aFraction handle.
>>> 5.714285714285714```#### Using `self` as an argumentTo make our bindings more object-oriented, the next step is to move the behaviour manipulating our objects closer to those objects.In other words, define our call-outs in the classes they manipulate, e.g., ask a fraction to transform itself into double.```language=smalltalkaFraction asDouble.
>>> 5.714285714285714```A naïve, yet working, implementation of such a binding would be to simply move our already-existing methods to the `FractionStructure` class:```language=smalltalkFractionStructure >> asDouble [
  ^ self fractionToDouble: self
]

FractionStructure >> fractionToDouble: aFraction [
  ^ self ffiCall: #(double fraction_to_double(FractionStructure* a_fraction))
]```However, uFFI allows us to enhance our bindings even further, combining external objects with `self` arguments in our call-outs.Indeed, our `asDouble` and `fractionToDouble:` methods can be merged into a single method using `self` as a literal argument to the function.```language=smalltalkFractionStructure >> asDouble [
  ^ self ffiCall: #(double fraction_to_double(FractionStructure *self))
]```### Opaque ObjectsMany libraries hide their internal representation using opaque data-types.An opaque data-type is data-type whose internal representation is not exposed to us.We can think of it as a structure whose fields are not visible to us, with the caveat that it may not actually be a structure.Libraries using such data-types restrict users to create values of such types and manipulate them only through the library's own functions, rendering an API similar to encapsulated objects.uFFI provides support for opaque objects through the `FFIOpaqueObject` class.#### Defining an opaque typeConsider an external function which publishes its public API through a header file with function definitions.This header file defines a type `fraction` although we do not know how it is internally defined.```language=ctypedef struct str_fraction fraction;
fraction mk_fraction(int numerator, int denominator);
float fraction_to_float(fraction f);```The simplest way to map such definitions is through type aliases:```language=smalltalkFFITutorial class >> initialize [
	fraction := #FFIOpaqueObject.
]

FFITutorial class >> makeFractionFromNumerator: n denominator: d [
	self ffiCall: #(fraction mk_fraction(int n, int d)).
]

FFITutorial >> fractionToFloat: aFraction [
	self ffiCall: #(float fraction_to_float (fraction aFraction))
]```#### Opaque types are External ObjectsTo make the example above more object-oriented, opaque data-types can be easily defined as external objects with the `FFIOpaqueObject` class. An `FFIOpaqueObject` is an external object that assumes nothing about its internal representation: it's just a pointer to some external data.Moreover, we can then define bindings inside that class, and use `self` as an argument to simplify our bindings.```language=smalltalkFFIOpaqueObject subclass: #OpaqueFraction
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'FFITutorial'
  
OpaqueFraction class >> makeFractionFromNumerator: n denominator: d [
	self ffiCall: #(OpaqueFraction mk_fraction(int n, int d)).
]

OpaqueFraction >> toFloat [
	self ffiCall: #(float fraction_to_float (self))
]```### ConclusionIn this chapter we have seen two different strategies for mapping external libraries: façades and external objects.Façades are the simplest approach, and pretty straightforward for simple or small libraries: they are god-like objects containing all the call-out definitions of our bindings.External objects allow us to distribute our bindings in a more object-oriented fashion, putting the behaviour closer to the data, and exploiting encapsulation.Complex data-types, as defined in the previous chapters \(unions, structures, enumerations\), are external objects in this sense. Moreover, we saw that we can define **complex data-types** using opaque objects and type aliases.These two strategies are not incompatible, so the same library mapping can mix-and-match, extracting functions into external objects as desired.