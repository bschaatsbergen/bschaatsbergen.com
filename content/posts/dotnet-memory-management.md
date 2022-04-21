---
title: "Memory management in .NET"
date: 2022-04-21T14:15:44+01:00
tags: [".NET"]
---
This article provides a kick-start to become more familiar with the memory mangement concepts
throughout .NET. In addition to this, we'll talk about memory fundamentals and best practices for managing and profiling/tracing your application its memory. This will help you to understand what the CLR and Garbage Collector are doing for you.


## Memory

It's important to note that there's 'managed code' and 'unmanaged code'. To put it very simple,
managed code such as C# code is code whose execution is managed by a runtime (CLR in this case).
The CLR takes responsibility for compiling it down into machine code and executing the instructions on the CPU. Besides that the CLR is also responsible for sub routines during the execution of the code, think of: automatic memory management, security, thread and type safety.

Opposed to managed code, unmanged code is something you can find in C/C++ programs. This code is completely  delegated and orchestrated by the developer. The result of the written code is just a set of binary instructions that the OS loads into memory.

Most high-level languages are garbage collected which means that a form of automatic memory management is running constantly active.


Every application has its own Virtual Address Space, these processes share the same Physical Memory through an intermediate page file. To reserve memory, the garbage collector calls the Windows API. The garbage collector also reserves segments, and releases segments back to the OS.


Virtual memory can be in three states:
- Free
  - Memory block has no references and is available for allocation.
- Reserved
  - Memory block is available and cannot be used for any other allocation request. You cannot store data to this block until it is committed.
- Committed
  - Memory block is assigned to physical storage.

You'll never manipulate physical memory directly, the Garbage Collector allocates and frees
virtual memory for you on the managed heap. If you ever have to work with the virtual address space I recommend taking a look at the [memoryapi.h](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi) which is part of the Windows API and has functions such as <code>VirtualAlloc</code> to reserve, commit or change the state of a page in the virtual address space.


![Virtual to Physical](/virtual-to-physical.png)


It's possible to run into a [OutOfMemoryExceptions](https://docs.microsoft.com/en-us/dotnet/api/system.outofmemoryexception?view=net-6.0) if either there's not enough Physical memory
space to commit or if there's not enough virtual address space available to reserve. (For .NET applications running on x86 processors, the virtual address space has a 2GB range 0x00000000 through 0x7FFFFFFF, in opposition to x64 which has a virtual address space ranging from 0x000'00000000 through 0x7FFF'FFFFFFFF making it 128TB).


### Memory allocation

When you start your application a new process is initialized, the runtime reserves a region of virtual address space for the process. There is a heap for each managed process.
This space is called the managed heap, think of the Heap as a place for dynamic memory allocation. The managed heap maintains a pointer to the next object that
is allocated on the heap, reference types are stored here. The first reference object that's stored on the heap is stored in the base address of the virtual address space.
When the next object is supposed to be stored on the heap, the Garbage Collector allocates memory for it in the next available address following the last stored object. As long as there's address space available in the heap the garbage collector will continue to allocate space and assign new objects to memory addresses on the heap.

Large objects that surpass 85,000 bytes are stored on the Large Object Heap (LOH),
the other objects are stored on the Small Object Heap. It's rare for objects to be so large, primarily arrays are held onto the LOH.

In .NET 5 the GC Team released a new segment in the Heap called the Pinned Object Heap, I recommend reading up on this yourselves. Problem/Solution is described here: [PinnedHeap.md](https://github.com/dotnet/runtime/blob/main/docs/design/features/PinnedHeap.md)
and the internals of the Pinned Object Heap are described here: [Internals of the POH](https://devblogs.microsoft.com/dotnet/internals-of-the-poh/).


I like to refer to the 'Stack' as a scratch pad for a running thread where scoped code is defined opposed to the Heap that stores data that is accessible throughout the application. Memory on the stack is known at compile time, think of it as static memory. Be aware that every thread has it's own instance of a Stack. When a function is called,
address space is reserved on top of the stack in the so called (StackFrame). This is to store local variables and 'meta' data such as MethodBase information, etc. Equal to the Stack data structure, the stack is LIFO (last-in first-out). This makes it fairly easy for the runtime to keep track of what what address space needs to be freed, this is as simple as adjusting one pointer.

![Stack and Heap](/stack-heap-example.png)

The preceding image shows where the code is accessed from, as <code>Person</code> is a POCO (reference type) it resides on the Heap, opposed to the <code>Guid g</code> that's assigned to the <code>Id</code> property from the <code>Person</code> instance. The <code>Guid g</code> is a local variable that's freed after the thread leaves the function block, therefor refering to the stack as a scratchpad for the executing thread is a good example.

It's good to note that the Stack is genuinely faster as the access patterns that are used to allocate and deallocate memory is just simply adjusting a pointer,
while the Heap has a much more complex pattern involved to record what's going on during allocation and deallocation. The heap is a global resource in a way that applications can refer from anywhere to reference types on that heap, after all that's why they are reference types.
Though this means that multiple threads can access these address spaces, which means that the CLR needs to take thread-safety into account when accessing objects on the heap.

Quoting a comment from Jeff Hill on reusability of bytes residing in the stack, which I came across on Stackoverflow:

<blockquote class="blockquote">"Each byte in the stack tends to be reused very frequently which means it tends to be mapped to the processor's cache, making it very fast." - Jeff Hill</blockquote>

## Garbage collection

Part of the CLR is the garbage collector (GC), this serves as a memory manager during the lifetime of an application. It's a very important feature for a high-level language such as C# as it takes away a 'burden' for developers. Best practices for memory management are now abstracted away in the runtime and worked on by professionals in that area of the .NET runtime.

### Releasing memory

For all we know now, memory is allocated and accesible through the Heap and the Stack, lets take a look at how the runtime releases memory.

The process of releasing memory is called garbage collection. The garabage collector determines when it's time to perform a collection based on the allocations being made.
When the collection is made it sorts out which objects are not used by the application anymore and it frees them (e.g. a string that's instantiated in the scope of a function, it's a reference type so therefor residing in the heap, but as the variable can only be accessed during the execution of the method, the variable becomes no longer needed after execution of the method.) Application roots are used in order to determine if a reference type is ready to be freed from the memory. Each root refers to an object in the heap or is set to null.
The garbage collector keeps track of the roots and creates graphs per active root containing all the objects that are reachable for the garbage collector through the root.

From the Garbage Collector its point of view, the root is a reference to an object that must not and will not be collected. This makes roots the only possible starting point for building retention graphs. The roots iterate through all objects and marks anything in-use or with a reference to an in-use object as 'live'. Once all objects are known and tagged with 'live', remaining unreachable objects can be passed to the garbage collector and address space can be 're-used' for new objects (now I quoted re-used as it won't be re-used in a direct way, as free memory is always located at the end of the heap) Once the reachable objects have been compacted, the garbage collector changes pointers so that the roots point to the objects in their new locations. The garbage collector also takes care of positioning the heap's pointer after the last reachable object as explained earlier.

Understanding root types is extremely important during the "Who retains the object?" analysis. Sometimes examining retention paths does not give you an answer why the object is still in memory. In this case, it makes sense to look at GC roots. The GC has multiple root types, but I do not feel like going in detail on these as I rather recommend that you look into this yourself. Note that in release builds, root's life times may be shorter — JIT can discard the variable right after it is no longer needed.

To sum it up, memory in .NET is not so easy as it seems and it involves a complex graph of references and cross-references. This is a common trap for developers to find themselves a memory leak as you can easily get lost in an object being anchored to more than one root in a deep web of references.

![Garbage Collector roots](/gc-roots.png)

<br><br>

![Bruno service references](/brunoservice-refs.png)

In the preceding image I've made a visual of the roots at runtime, a method is called that's dependent on both a <code>memoryCache</code> and <code>redisCache</code>. The roots mark the referenced object as 'live' and exclude them from the collection.

In the diagram there's also a 'dereferenced' object that's pointing to an object that is marked as 'live', this is a great example of a memory leak as the application won't release this memory unless the dependent object which the dereferenced object is referenced too is set to null. Tracing memory leaks can be very though as you need to be aware of the leak first (which is when symptoms of rising memory usage are becoming noticeable), but then what? I like to use dotMemory when analyzing memory states, when comparing the states by making snapshots it is possible that you find the unused objects.

In dotMemory you can take a snapshot at 2 points during the lifespan of your application to compare them, the comparison view shows a detailed overview of objects that were created between the 2 points, e.g. if a class should not have multiple instances but in the comparison view it shows it does.

![Bruno comparison](/bruno-comparison.png)

Here it becomes clear what happend, a memory leak involving one of the POCOs (Plain Old CLR Objects) in my application. Appareantly something is creating new instances and adding them to a List which is not released. By looking into the objects that reside in the list we can indeed confirm the assumption, unnecessary memory is allocated within the virtual address space of the process without being released properly.

![Bruno instances](/bruno-instances.png)

The amount of times that the garbage collector frees memory is dependent on the volume of allocations being made on the heap and memory that was held onto the heap from previous garbage collection invocations. In language ecosystems where we have automatic memory management but yet explicitly invoke the garbage collector ourselves we should raise a serious red flag as it can prevent the garbage collector from doing it's work properly.

### Garbage collector generations

The garbage collector is divided in three generations: 0, 1 and 2.
Each gen has it's own responsibility and takes care of different objects.
The garabage collector is responsible for moving the objects between the different gens.
Newly created objects are always stored in gen 0 by the garbage collector (however if they exceed the LOH limit, they are stored on the large object heap). Objects created early in the
application and those that survive collections (think of registering services to the DI container) are promoted to gen 1 or 2 and stored there. Because of the way the garbage collector 'frees up' memory by compacting it, it's more efficient to compact a partition of the heap instead of the entire heap. If this does not reclaim enough memory the garabage collector performs a gen 1, followed by a gen 2 clean up if needed (which is fairly rare as most gen 0
collections reclaim enough memory). Objects that survived the gen 1 collection are immediately promoted to gen 2 in this case. User code can only allocate in generation 0 or the Large Object Heap.

Most objects in gen 0 are collected by the garbage collector and don't survive a promotion.
But what happens if gen 0 is full and you are trying to allocate a new local scoped string? It will trigger  gen 0 collection in an attempt to free virtual address space to allocate the string on gen 0.

Objects that survive a gen 0 collection are promoted to gen 1 objects, gen 1 is referred to
as a buffer between short and long-living objects. Surviving a collection indicates that an object is deemed to live longer on the heap.

Gen 2 objects are long-living objects, think of singletons and static lists defined at the top of a class. Objects in gen 2 that survive a gen 2 collection will persist until an application root marks them as unreachable. Objects in the Large Object Heap (LOH) are collected during gen 2 collections.

It's important to note that each collection on gen N also performs a collection on any gen below gen N. Gen 2 collections are typically reffered to as a full garbage collections.

### Unmanaged resources paradigm

Now it's good to have taken a look under the hood of the CLR and how the garbage collector saves us manual memory management.
But can high-level languages that are garbage collected still somehow have unmanaged resources? Well for most object instances you can rely on the work that the garbage collector does. However there can be unmanaged resources which require explict memory releases, these are outside of the domain of the runtime. The garbage collector does not allocate or release unmanaged memory, this is also where the need for the dispose pattern arises from.

The most common type of unmanaged resources are object instances that wrap around OS components, such as files and network connections.
Unmanaged resources can be cleaned up in a various ways but often we call Dispose or use a Using block.
The purpose of the Dispose method is to release unmanaged resources held by an object. Freeing
the actual memory is done by the garbage collector. Note that a Using statement calls the Dispose method if the object you're dealing with
implements it, if the object implements IAsyncDisposable the Using statement calls for DisposeAsync and awaits the returned Task. The compiller translates the Using statement in a try/catch under the hood so that the Using statement makes sure that Dispose/Async is called even if an exception raises within the Using block. For more info: [Using objects that implement IDisposable](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/using-objects).
