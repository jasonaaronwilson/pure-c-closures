# Pure C Closures

This ia an outline of how one might do closures in pure C (potentially with some macros to make it not so ugly). We will not try to support variable number of arguments for these closures.

If you are simply compiling to C and don't need to pass your closures to C or C++ code, then you can do something much simpler than what I propose here, i.e., something along the lines of storing a function pointer in slot zero and the rest of the data in other slots of a normal memory region (struct). If you know when you are calling a closure, you can emit the slightly more complicated code to first indirect on the closure pointer to obtain the closure's body function pointer, and then invoke this body pointer with the closure pointer as the first arguement and the user arguments as the remaining arguments.

In case where you want to pass a closure as a callback in regular C code (say a library) that doesn't know about how our closures are represented, then we need a closure pointer to behave like a function pointer! I believe gcc can do this via dynamic code generation of a trampoline but this isn't standard C and so may not be available with older C compilers.

However the idea of using trampolines to invoke the actual body of the closure can still work if we merely have enough statically compiled trampolines to hold as many live closures of each type that as might be needed. (We will tackle the static part later by loading a .so file multiple times, maybe something you wouldn't do for embedded development for example).

```
extern int closure_body_abc(uint64_t data_id, int a, int b) {
   struct foo* data = runtime_lookup_closure_data(data_id);
   if (data == NULL) {
      /* ERROR */;
   }
   /* closure body goes here */
}

extern int trampoline_abc_00(int a, int b) {
   return closure_body_abc((int64_t) &trampoline_abc_00, a, b);
}
extern int trampoline_abc_01(int a, int b) {
   return closure_body_abc((int64_t) &trampoline_abc_01, a, b);
}
/* Add enough trampoline's so that we have enough to handle the maximum
   number of "live" closures for "abc".
*/
```

We need additional machinery of course. Presumabldy runtime_lookup_data is a call to some hashtable lookup function. We also need runtime_allocate_closure and rutime_free_closure. So we need to add a free list for each kind of trampoline and somehow initially populate the free list. There are potentially faster solutions like directly mapping data pointers to an array, using a bitmask for free vs allocated, etc.

We can now determine some costs for these closures. Clearly we've added at least one and maybe two extra calls and returns and of course the hash-table lookup code this is probably some number of microseconds. So the over-head is definitely going to be felt for closures that would otherwise run pretty quickly. However if the closure itself is kind of hefty, then maybe this approach isn't too bad. Obviously there is a cost associated with merely having the extra trampoline's in your binary. The best way to measure this will be be creating a working example and compiling it on several platforms. The more arguments, likely the larger each trampoline will be.

## "Unlimited" Closure Counts

Now that we understand the costs a little better, let's address the elephant in the room: the need to determine statically the number of trampoline's to have in the binary. This problem can be worked around somewhat, again in pure C (though not necessarily Posix) by putting trampolines of a particular type into a shared library and simply loading this shared library at a free address to allocate N more trampolines of that type! This won't be great for tail latency as loading a shared library is somewhat expensive (I imagine) and there may be issues of having loaded too many shared libraries (not sure what the real limit would be), but it should get the job done!

If we are willing to embrace more particular properties of the compiler, we might be able to just copy a function from memory to an executable part of the heap and have truly dynamic counts with less contraints. The key here is to use PIC code when compiling the files containing the trampolines. In these cases we would expect most compilers to use some kind of PC relative construct to get the address of "self" before calling the actual body. We can probably tell if this technique would work with common C compilers simply be disassembling code compiled with those compiler's on their supported platforms. We might need to turn off all or some optimizations.

# Conclusion

This is a thought experiment in how to represent closures in pure C so that these closure can be called by C and C++ as though they are merely function pointers. Instead of run-time code generation, we let the compiler create trampolines and manage them as a limited resource at run-time. If we truly need dynamic counts for some of the closures, we can use the shared library technique to load more libraries.
