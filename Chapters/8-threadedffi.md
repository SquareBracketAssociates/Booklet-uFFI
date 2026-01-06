## Non-Blocking Calls

This chapter presents how to use the non-blocking feature of uFFI.
This feature allows to only block the Pharo process doing the FFI call.
We start with some concepts and clarifications, then we present how to use the non-blocking calls,
 and finally, we talk a little about the impact with callbacks.

### Definitions

For a more clear description of the _Non-Blocking calls_ we need to define some concepts:

- OS Thread: These are the threads provided by the operating system. They can run in parallel and concurrently depending of the scheduling of the OS. 

- VM Execution Thread: This is the thread responsible for executing the Pharo code. All Pharo processes run in this thread. Also, all the operations of the VM run in this thread e.g., Snapshotting, Garbage Collect, and JIT Compiler.

- Pharo Process: a process is the multiprocessing unit of Pharo. Processes are green threads handled by the VM. At all times, multiple processes are running at the same time. They are planned and executed in the VM Execution Thread.

- Blocking Calls: A blocking FFI call, is a call that is executed directly in the VM Execution Thread. This call blocks the execution of all Pharo code. Performing these calls is cheap and fast, but if the FFI call is long will block the whole execution of the VM, and all Pharo processes with it.

- Non-Blocking Calls: A non-blocking FFI call is executed in an FFI Worker thread. This call just blocks the worker thread and the Pharo process performing the call. The VM Execution Thread continues executing other Pharo Processes. This type of call allows us to execute long-running FFI calls without freezing the whole Pharo image. However, performing a Non-Blocking call is expensive as it is required to synchronize and communicate two OS Threads, the one of the VM Execution and the one of the FFI Worker.

- FFI Runner \(**TFRunner**\): A FFI Runner is the object responsible for handling the FFI Calls and Callbacks. It implements different strategies to run them. 

- FFI Worker Thread \(**TFWorker**\): This FFI Runner uses an OS thread to execute FFI calls. It has an input queue where outgoing FFI calls are put and a response queue that notifies the VM Execution Thread when an FFI call has finished. A FFI Worker Thread performs only one call at a time. If a call is being executed, it is blocked in this call. If it is required to execute more than one call at a time in the worker, a second worker should be created. There is always a default FFI Worker thread that is available, but additional ones can be created and referred them by a name. 

- Same Thread Runner \(**TFSameThreadRunner**\): This FFI Runner is the default FFI Runner used by the FFI Calls. It performs blocking calls using the VM Execution Thread. It is fast and performant to use with short FFI calls.

- Main Thread Runner \(**TFMainThreadRunner**\): This FFI Runner is a special case of an FFI Runner. It works exactly as a FFI Runner with the difference that uses the main thread of the OS Process to run the FFI calls. It is useful when we require that FFI calls be performed in the main thread of the application. This is a common require of UI frameworks e.g., Cocoa. This Worker is only available if the Pharo VM is run with the _--worker_ option.

### Using a Worker Thread to execute FFI calls

The selection of the FFI Runner to use is FFILibrary dependent. 
Each library will say which FFI Runner to use. By default all libraries are using the **TFSameThreadRunner**.
To select to use the default FFI Worker Thread we should override the method **#runner**. 
For using the default TFWorker we will do:

```language=smalltalk
runner

	^ TFWorker default
```

If we want to use a different one just for this library we can use:

```language=smalltalk
runner

	^ TFWorker named:'myLibrary'
```

A same worker can be shared by many libraries, but only one call will be performed at a time.

### Mixing Same threads calls and Worker Thread calls

If we need to mix same thread and worker thread calls, an easy way is to have a subclass of the library that overrides the **runner** method. As an example, we will have the **FFILibrary** subclass called **MyFFILibrary** with all the information to lookup the dynamic libraries and the configuration that we need. **MyFFILibrary** will use the default **#runner** implementation that returns **FFISameThreadRunner**, so it will be used for fast blocking calls. Also, we will have a subclass of **MyFFILibrary**, **MyFFILibraryUsingWorker** that just reimplemnets the **#runner** method returning the default **TFWorker** instance. By doing so, FFI calls using **MyLibrary** will be fast blocking, and the ones using **MyFFILibraryUsingWorker** will be non-blocking.

Next, we need to select which library to use, for doing so we are going to use the **#ffiCall:library:** selector instead of **#ffiCall:**. For example having a blocking call to **myShortFunction** will be like

```language=smalltalk
myFunction

	^ self ffiCall: #(void myShortFunction()) library: MyLibrary 
```

And a non-blocking version to a long running function **myLongFunction** is :

```language=smalltalk
myFunction

	^ self ffiCall: #(void myLongFunction()) library: MyFFILibraryUsingWorker 
```

### Selecting FFI Calls to execute in a Worker

As said before, using a worker thread to execute a FFI call has a performance penalty. 
Worker Thread calls are around 20 times slower than calls performed with the Same Thread runner.
The developer is responsible of selecting the FFI calls that are worthy of being executed in a Thread Runner.
An easy metric to determine this could be average time taken by the called C function. 
If the called C function takes more than some hundreds of milliseconds it will be useful to perform in a worker thread.

### Non Thread Safe Libraries

If the C library called using FFI is not thread safe, we need to guarantee that all the calls to it are performed in the same thread. In this case, it is not possible to mix calls in different runners. 
The developer is responsible to know if the library can be called from different threads.

A thread safe library not only will allow us to mix fast blocking and non-blocking calls, but also it allows us to have different worker instances implementing concurrent calls.

### Callback considerations

To correclty handling the flow of FFI calls and the C stack, callbacks should be handled in the same thread of the library.
This is automatic when we are using the Same Thread Runner. However, when using a Worker thread the FFICallback should have the correct FFILibrary assigned to it.

To give the correct library to a callback we should set it through the **#ffiLibrary:** message after creating the callback and before passing it to the C library.

If we are using anonymous callbacks we should create the callback with the message **#newCallbackWithSignature:block:library:**. 