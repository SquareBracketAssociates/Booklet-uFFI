### Handles \(Windows\)
	HWND := #FFIConstantHandle.
]

User32 class >> getActiveWindow [
	^ self ffiCall: #(HWND GetActiveWindow())
]
	| result address typeClass |
	
	result := OrderedCollection new.
	address := ExternalAddress new.
	typeClass := FFIExternalType resolveType: aTypeName.
	[ self iterator_next: address ] 
		whileTrue: [ result add: (address castTo: typeClass) ].
  ^result
 ]

Iterator >> iterator_next: data [
	^ self ffiCall: #(Boolean iterator_next (Iterator self, void** data))
]
	^ aClass fromHandle: self
]
	^ aClass valueFromHandle: self
]

FFIExternalType class >> valueFromHandle: anAddress [
	“This is used for structures, but we can reuse them :)"
	^ self new handle: anAddress at: 1 
]
		
FFIExternalReference class >> valueFromHandle: anAddress [
	^ self fromHandle: anAddress
]

FFIExternalStructure class >> valueFromHandle: anAddress [
	^ self fromHandle: anAddress
]