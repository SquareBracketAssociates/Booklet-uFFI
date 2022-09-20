## Marshalling, Types and Function Arguments
	^ self ffiCall: #( int abs ( int n ) )
]
	^ self ffiCall: #( int abs ( int anInteger ) )
]
=> 42
=> 42
=> 1073741823
=> 1
=> Error: Could not coerce arguments
=> 3
=> -2147483648  "in 32-bit Pharo images"
=> 2007355392  "in 64-bit Pharo images"
=> Error: Could not coerce arguments
=> -2147483648
	^ self ffiCall: #( int abs ( float aFloat ) )
]
=> 1082130432  "in 32-bit Pharo images"
=> 0  "in 64-bit Pharo images"
	^ self ffiCall: #( float abs ( int anInteger ) )
]
=> Float nan  "in 32-bit Pharo images"
=> -1.07374176e8  "in 64-bit Pharo images"
	^ self abs: -42
]
	^ self ffiCall: #( int abs ( int -42 ) )
]
	...
	classVariableNames: 'MagicNumber'
  ...

FFITutorial class >> initialize [
	"Set this to -42 because.. Life, the Universe, and Everything."
	MagicNumber := -42.
]
  ^ self ffiCall: #( int abs ( int MagicNumber) )
]
	...
	classVariableNames: 'MagicNumber'
  ...

FFITutorialPool class >> initialize [
	"Set this to -42 because.. Life, the Universe, and Everything."
	MagicNumber := -42.
]

Object subclass: #FFITutorial
	...
	poolDictionaries: 'FFITutorialPool'
  ...

FFITutorial class >> absMinusFortyTwo [
  ^ self ffiCall: #( int abs ( int MagicNumber ) )
]
	^ self ffiCall: #( int myFunctionNameInstVar ( int myArgumentInstVar ) ) library: LibC
]
	^ self ffiCall: #( int abs ( int self ) ) library: LibC
]
	^ self ffiCall: #( void * malloc ( size_t aSize ) ) library: LibC
]
 => (void*)@ 16r7FFBDE0DE030
 => (void*)@ 16r00000000
	^ self ffiCall: #( void free( void *ptr ) ) library: LibC
]

FFITutorial free: anExternalAddress.
  => @ 16r00000000
	^ self ffiCall: #( void * realloc ( void* aPointer,  size_t aSize ) ) library: LibC
]
 => (void*)@ 16r7FFBDE0DE030
 => (void*)@ 16r7FFBDE0DE030
  int input=INT_MAX;
  printf("Overflowing int: %d + 1 = %d\n", input, input + 1);
}

void overflowing_uint(){
  unsigned int input=UINT_MAX;
  printf("Overflowing unsigned int: %u + 1 = %u\n", input, input + 1);
}
Overflowing unsigned int: 4294967295 + 1 = 0
 => LargePositiveInteger
  //Both floating point numbers should have the same value
  float x = 3.141592653589793238;
  double y = 3.141592653589793238;
  
  printf("Float is: %20.18f\n", x);
  printf("Double is: %20.18f\n", y);
}
Double is: 3.141592653589793116
	printf("Boolean: %d\n", b);
}
	^ self ffiCall: #( void printBoolean ( bool b ) )
]