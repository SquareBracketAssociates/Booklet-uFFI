## Designing with uFFI and FFI Objects
  self ffiCall: #( int abs (int n) ) library: MyLibC
]
  self ffiCall: #( int abs (int n) )
]

FFITutorial >> ffiLibrary [
  ^ MyLibC
]
  ^ self
]
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
]
  instanceVariableNames: 'handle'
  classVariableNames: ''
  package: 'FFI-Kernel'
{
	int numerator;
	int denominator;
} fraction;

double fraction_to_double(fraction* a_fraction){
  return a_fraction -> numerator / (double)(a_fraction -> denominator);
}
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'FFITutorial'

FractionStructure class >> fieldsDesc [
	^ #(
		int numerator;
		int denominator;
		)
]
  ^ self ffiCall: #(double fraction_to_double(FractionStructure* a_fraction))
]

FractionStructure rebuildFieldAccessors.
aFraction := FractionStructure externalNew.
aFraction numerator: 40.
aFraction denominator: 7.
FFITutorial new fractionToDouble: aFraction.
>>> 5.714285714285714
  ^ self ffiCall: #(double fraction_to_double(void* a_fraction))
]

...
FFITutorial new fractionToDouble: aFraction handle.
>>> 5.714285714285714
  fraction* f = (fraction*)malloc(sizeof(fraction));
  f -> numerator = numerator;
  f -> denominator = denominator;
  return f;
}
  ^ self ffiCall: #(FractionStructure* make_fraction(int numerator, int denominator))
]

aFraction := FFITutorial new newFractionWithNumerator: 40 denominator: 7.
FFITutorial new fractionToDouble: aFraction.
>>> 5.714285714285714
  ^ self ffiCall: #(void* make_fraction(int numerator, int denominator))
]

aFraction := FractionStructure fromHandle: (FFITutorial new newFractionWithNumerator: 40 denominator: 7).
FFITutorial new fractionToDouble: aFraction handle.
>>> 5.714285714285714
>>> 5.714285714285714
  ^ self fractionToDouble: self
]

FractionStructure >> fractionToDouble: aFraction [
  ^ self ffiCall: #(double fraction_to_double(FractionStructure* a_fraction))
]
  ^ self ffiCall: #(double fraction_to_double(FractionStructure *self))
]
fraction mk_fraction(int numerator, int denominator);
float fraction_to_float(fraction f);
	fraction := #FFIOpaqueObject.
]

FFITutorial class >> makeFractionFromNumerator: n denominator: d [
	self ffiCall: #(fraction mk_fraction(int n, int d)).
]

FFITutorial >> fractionToFloat: aFraction [
	self ffiCall: #(float fraction_to_float (fraction aFraction))
]
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'FFITutorial'
  
OpaqueFraction class >> makeFractionFromNumerator: n denominator: d [
	self ffiCall: #(OpaqueFraction mk_fraction(int n, int d)).
]

OpaqueFraction >> toFloat [
	self ffiCall: #(float fraction_to_float (self))
]