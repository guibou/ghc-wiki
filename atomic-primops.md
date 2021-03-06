# Atomic primops



Atomic operations on basic numeric (and pointer) types can be used as a foundation to build higher level construct (e.g. mutexes.) There are also useful on their own (e.g. to implement an atomic counter.) This page outlines a design for adding atomic primops.


## The primops



The new primops are modeled after those provided by C11, C++11, GCC, and LLVM.


```
atomicReadIntArray# :: MutableByteArray# s -> Int# -> State# s -> (# State# s, Int# #)
atomicWriteIntArray# :: MutableByteArray# s -> Int# -> Int# -> State# s -> State# s
fetchAddIntArray#
    :: MutableByteArray#     -- Array to modify
    -> Int#                  -- Index, in words
    -> Int#                  -- Amount to add
    -> State# s
    -> (# State# s, Int# #)  -- Value held previously
fetchSubIntArray# :: MutableByteArray# -> Int# -> Int# -> State# s -> (# State# s, Int# #)
fetchOrIntArray#  :: MutableByteArray# -> Int# -> Int# -> State# s -> (# State# s, Int# #)
fetchXorIntArray# :: MutableByteArray# -> Int# -> Int# -> State# s -> (# State# s, Int# #)
fetchAndIntArray# :: MutableByteArray# -> Int# -> Int# -> State# s -> (# State# s, Int# #)
casIntArray# :: MutableByteArray# s -> Int# -> Int# -> Int# -> State# s -> (#State# s, Int##)
```


`fetchAddIntArray#` and `casIntArray#` already exist (but are implemented as out-of-line primops.)


## Implementation



The primops are implemented as `CallishMachOp`s to allow us to emit LLVM intrinsics when using the LLVM backend. This also allows us to provide a fallback implementation in C on those platforms where we don't want to implement the backend support for these operations. The fallbacks can be implemented using the GCC/LLVM atomic built-ins e.g. `__sync_fetch_and_add`.


### Ensuring ordering of non-atomic operations



A `CallishMachOp` already acts as a memory barrier; the Cmm optimizer will not float loads/stores past it. We'll document the reliance on this behavior in the sinking pass to make sure that if it's ever changed, the optimizer is taught about how to handle these atomic operations in some different way.



The `CallishMachOp` will be translated to the correct instructions in the backends (e.g. `lock; xadd` on x86).



As the Cmm code generator cannot reorder reads/writes around prim calls (i.e. `CallishMachOp`s) the `memory_order_seq_cst` semantics should be preserved, as long as the backend outputs a memory barrier to prevent CPU speculation.


## Ordering of non-atomic operations



We'll use sequential consistency, which corresponds to the C++0x/C1x `memory_order_seq_cst`, Java `volatile`, and the gcc-compatible `__sync_*` builtins. This is the strongest consistency guarantee. We can provide weaker guarantees in the future, if needed.


