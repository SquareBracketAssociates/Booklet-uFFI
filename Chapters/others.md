### Handles \(Windows\)A constant HANDLE, as described in [Windows MSDN](https://msdn.microsoft.com/en-us/library/windows/desktop/ms724457(v=vs.85).aspx)is a special kind of external object which is identified numerically. Therefore, an `ExternalAddress` is not appropriate to describe it \(since these handles are constants and an external addresses represents disposable spaces in memory\).It is not clear that this is necessary outside of Windows, but according to the documentation they are somewhat analogous to unix's File Descriptors \(with some remarkable diferences, as documented [here](http://lackingrhoticity.blogspot.fr/2015/05/passing-fds-handles-between-processes.html).Example: ```User32 class >> initialize [
	HWND := #FFIConstantHandle.
]

User32 class >> getActiveWindow [
	^ self ffiCall: #(HWND GetActiveWindow())
]```### Annexes#### Casting In Pharo, “casting” as made in C is not necessary since this is just a way to tell the C compiler that a pointer is for a certain type \(but it is always a pointer, and in machine code it is always the same\). Nevertheless, there are some usages were we may want to see pointers as specific instances in our image. The next examples will cover these cases.##### Converting pointers to objects of any typeSo, here you will do something like this: Let's suppose you want to execute something like this \(taken from a question on the pharo mailing list\): ```Iterator >> asCollectionOfType: aTypeName [
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
]```That is, you want to iterate over a collection of a certain type and convert it to proper instances on the Pharo side. While you could always simply implement `#castTo:` like this: ```ExternalAddress >> castTo: aClass [
	^ aClass fromHandle: self
]```But you will still have a problem if your type is an “atomic” type, like `int`, `long`, etc. because those values are passed “by value” and not by pointer, so you will need to decode them. In this case, I would implement a method extension and use double dispatch: ```ExternalAddress >> castTo: aClass [
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
]```Then your cast will work as expected.##### Non conventional castsCasting in C is trivial. You can do something like this:```void *var = 0x42000000.```And you will be creating a pointer who points to the address 0x42000000. These kind of declarations are used in certain frameworks, notably some Windows libraries.In Pharo, this "casting" is not so easy, and we need to declare these kinds of variables as ExternalAddresses. We do this like this:```var := ExternalAddress fromAddress: 16r42000000.```## FAQsIf someone is interested to collect questions and answers from the mailing-list this is where there could go. 