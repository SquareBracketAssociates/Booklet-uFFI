## uFFI and memory management
  // Automatic variable allocated in the stack
  int t = 42;
}
  int* pointer = (int*)malloc(17);
  //Free that memory
  free(pointer);
  instanceVariableNames: 'x y'
  classVariableNames: ''
  package: 'Kernel-BasicObjects'

Point new
  instanceVariableNames: ''
  classVariableNames: ''
  package: 'Collections-Sequenceable-Base'

Array new: 20.
" ... use my structure ... "
myStructure free.
myStructure autoRelease.
" ... use my structure ... "
" ... dereference it so it will be collected ..."
myStructure := nil.
  	handle isNull ifTrue: [ ^ self ].
  	handle free.
  	handle beNull
]
   handle isNull ifTrue: [ ^ self ].

   "Logging the handle in the transcript for information"
   ('Freed ', handle asString) traceCr.

   handle free.
   handle beNull
]
	^ self getHandle
]
	^ {self getHandle. self windowID }
]

SDL_Window class >> finalizeResourceData: aTuple [
  | handle windowId |

  handle := aTuple first.
  handle isNull ifTrue: [ ^ self ].

	windowId := aTuple second.
  OSSDL2Driver current unregisterWindowWithId: windowId.
  self destroyWindow: handle.
  handle beNull
]
" ... use my structure ... "
" nil it and PLUF, eventually the object will be discarded "
myStructure := nil.
  ^ self ffiCall: #(void myFunction(MyStructure* aStructure))
]
myStructure pinInMemory.
" ... safely use my structure ... "
" nil it and PLUF, eventually the object will be discarded "
myStructure := nil.