## Callbacks
	char* name;
	int age;
}

struct Animal myAnimals[10];
	if(oneAnimal->age == anotherAnimal->age)
		return 0;
	
	if(oneAnimal->age < anotherAnimal->age)
		return -1;

	return 1;
}
                  int (*comparisonCallback)(const void *, const void *));
	signature:  #(int (const void *arg1, const void *arg2))
	block: [ :arg1 :arg2 | ((arg1 doubleAt: 1) - (arg2 doubleAt: 1)) sign ].
	self
		ffiCall: #(void qsort (FFIExternalArray array, size_t count, size_t size, FFICallback compare)) 
		module: LibC
]