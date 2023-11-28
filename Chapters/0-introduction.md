## Introduction / Preface


This booklet contains a guide to the unified FFI framework \(uFFI\) for the Pharo programming language.
The aim of this booklet is not to be just an API reference, but to thoroughly document uFFI in an ordered way for both beginner and advanced users.
We present the different concepts of uFFI, from simple call-outs and marshalling up to how to memory is managed, all chapters including examples of code and/or pictures to illustrate those concepts.

You'll find the source code of this booklet stored in [https://github.com/SquareBracketAssociates/Booklet-uFFI](https://github.com/SquareBracketAssociates/Booklet-uFFI), where it has a companion bug tracker and list of releases.
As of this first edition \(v1.0.0\), we cover the uFFI framework as it exists in Pharo 8.0, released in january 2020.
Future editions will update this booklet for future Pharo versions, and document new features as they appear.
Do not hesitate to open an issue if you find a problem.

Enjoy your reading,
Guille


### What is FFI?


A Foreign Function Interface \(FFI\) is a programming language mechanism that allows software written in one language to use resources \(functions and data structures\) that were written in and compiled with a different language.  These "foreign" resources typically take the form of shared libraries, such as DLL files in Windows \(or '.so' files in Linux and Mac\) and can include run-time services made available by your operating system.  A good example is a driver library provided by a vendor of a computer peripheral, such as a printer or a network card.

Code sharing and reuse via an FFI is fundamentally different from source code "includes", calls to IDE libraries, or messages sent to the base classes provided by your language's development environment.  In those cases, your compiler can link function calls and share structured data while making convenient assumptions about the object code interface \(known as the ABI, or Application Binary Interface\).  This is familiar programming: We follow published APIs \(Application Programming Interfaces\) as we write our code, and the compiler takes care of the low-level connections for us transparently.

However, if the compiled resources to which we wish to link were built with a different language \(or even a different compiler of the same language\), the bits and bytes may not line up properly when we push arguments, make calls, and retrieve results, due to the different standards that the other code's compiler followed when the borrowed code was produced.  In this case, more detailed bookkeeping must be employed by our language to make ABI translations and ensure that differences in data alignment, byte ordering, calling conventions, garbage collection, pointer references, and so forth are all accounted for.  Without this, we risk not only getting incorrect results -- we could crash our process \(or even the operating system\).

While in theory FFI interfaces can be defined to link any pair of differing languages, most languages are designed to interact with libraries written in the C language.  This is because C has a long heritage and widespread use, and \(importantly\) C has a predictable, standardized way to compile functions and structures.  So with Pharo also choosing C as an FFI target we gain two major advantages: first, by basing ourselves on such "standard" formats, we can build tools that simplify interoperability with an extensive array of existing C libraries.  Second, since nearly every other language is doing the same, we can simply join those mutual C-standard ABIs together to form a bridge between different systems -- allowing us to program at the source level of a C API.

### Pharo, FFI and uFFI


From time to time it happens that we need to access a feature that is not available in Pharo's standard libraries, nor in a community package.
In such cases, we have the choice of implementing such a feature from scratch in an entirely Pharo-written library, or to reuse some existing implementation in another language.
To choose a strategy, one may consider many criteria such as personal taste, the ability to cope with the effort of implementing a new library from scratch, or the maturity, documentation and community of the existing libraries doing the job.
Without FFI, the option of reusing an external existing library would not exist, or it would be constrained to expert developers extending the execution interpreter with modules like virtual machine plugins.

This booklet shows the Unified FFI framework for Pharo \(uFFI for short\).
The Pharo uFFI is an API framework that eases communication between Pharo code and code that follows a C-style interface, making it possible to easily interact with external C libraries.  Given that 'calling functions' is the ultimate objective of someone using an FFI, you will see that uFFI also helps with several other concerns, such as finding C function interfaces, transforming data between the Pharo world and the C world, accessing C structures, defining C-to-Pharo callbacks, and others.


### How to read this booklet


We recommend beginners to read the booklet in order.
Chapter 2 is a good introduction to uFFI usage, introducing the concepts of callout and marshalling.
Chapter 3 dives into the details of marshalling and its rules, which may be a bit overwhelming at the beginning, so do not hesitate to skip parts and come back to it when you have acquired more experience.
Chapter 4 explaines different complex types, and can be read as needed to deal with particular data types.
Chapter 5 discusses how developers can enhance and organize their uFFI library bindings, so experience using uFFI is assumed.
Chapter 6 discusses the issues of memory management between Pharo managed memory and C managed memory, which is important to avoid issues in the long term.

We invite those users already comfortable with FFI and/or uFFI to jump around as needed.

### Acknowledgments


Pharo uFFI has been originally developed by E. Lorenzano, and was based on many ideas on the NativeBoost framework by Igor Stasenko.
uFFI has received many contributions over the years from Pharo's open source community.
The main backend of uFFI up to this day is the FFI implementation of the OpenSmalltalk VM, with contributors such as Eliot Miranda.
This booklet was mainly written by Guille Polito and Pablo Tesone, with extensive reviews and edits from Stéphane Ducasse, Ted Brunzie and Sean De Nigris.
