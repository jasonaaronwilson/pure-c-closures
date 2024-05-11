# Pure C Closures

This ia an outline of how one might do closures in pure C (potentially with some macros to make it not so ugly).

If you are simply compiling to C and don't need to pass your closures to C or C++ code, then you can do something much simpler than what I propose here, i.e., something along the lines of storing a function pointer in slot zero and the rest of the data in other slots of a normal memory region (struct). If you know when you are calling a closure, you can emit the slightly more complicated code to first indirect on the closure pointer to obtain the closure body pointer, and then invoke this body pointer with the closure pointer as the first arguement and the user arguments as the remaining arguments.

In case where you want to pass a closure as a callback in regular C code (say a library) that doesn't know about how closures are represented, then we need a closure pointer to behave like a function pointer! I believe gcc can do this via dynamic code generation of a trampoline but this isn't standard C and so may not be available with older C compilers.

However the idea of using trampolines to invoke the actual body of the closure can still work if we merely have enough statically compiled trampolines to hold as many live closures of each type that is needed. (We will tackle the static part later by loading a .so file multiple times, maybe something you wouldn't do for embedded development for example).

```
extern int closure_body_abc(uint64_t data_id, int a, int b) {
   struct foo* data = runtime_lookup_data(data_id);
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
