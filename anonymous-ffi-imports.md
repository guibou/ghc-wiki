
This page contains a proposal to add anonymous FFI imports to GHC.


## Background



FFI imports give a way to execute foreign code from Haskell.  Currently C and JavaScript foreign code is supported, as of GHC 7.8.



FFI imports are introduced as top-level definitions, e.g.


```wiki
foreign import ccall "cos" c_cos :: CDouble -> CDouble
```


However, it is often useful to call code directly, without a top-level declaration.  The obvious advantage is that we can call C/JavaScript libraries directly without defining bindings.  This approach is already taken by Fay, a Haskell-like language compiling to JavaScript, where we can say


```wiki
(ffi "foo(%1)" :: Text -> Char) "Hello!"
```


where `foo` is a JavaScript function.



This also makes writing libraries interoperating generically with foreign code much easier.  For example currently the `language-c-inline` library needs some inconvenient TH machinery to refer to the C functions it creates on the fly when processing a module with inline C.  This machinery would not be needed if we had anonymous FFI imports.



We propose to extend the language syntax so that anonymous FFI imports can appear in ordinary Haskell expressions, like so:


```wiki
(foreign import ccall “cos” :: CDouble -> CDouble) 1
```


Doing so should amount to no change past the desugarer.


## Syntactic extension



In `Parser.y`, a new alternative is added to the exp production rule:


```wiki
exp : ...
    | 'foreign' 'import' fimport

fimport : callconv safety fanonspec
        | calconv fanonspec

fanonspec : STRING '::' sigtypedoc
          |        '::' sigtypedoc
```


Where `fimport` and `fanonspec` are patterned against the already existing `fdecl` and `fspec`, respectively.  It might be worth thinking about factoring out the common parts between top-level and anonymous FFI.



This allows to use all the existing top-level FFI import facilities in an anonymous manner, by reusing the existing FFI import syntax without the naming.


## Typechecking extension



The code in `TcForeign.hs` already handles type-checking of top-level FFI imports, and said code can be imported wholesale with little modification to type-check anonymous FFI imports.  Obviously type-checking of anonymous FFI imports will not cause the binding of any name.


## Desugarer extension



FFI imports are already desugared to a ordinary Haskell functions.  For example the already mentioned


```wiki
foreign import ccall "cos" c_cos :: CDouble -> CDouble
```


Is desugared to (eliminating the core background noise from the `-ddump-ds` output)


```wiki
c_cos :: CDouble -> CDouble
c_cos =
  (\(x :: Double) -> D# $ {__pkg_ccall_GC main cos Double# -> State# RealWorld -> (# State# RealWorld, Double# #)} x realWorld#)
    `cast`
  (Sym CDouble -> Sym CDouble :: ((Double -> Double) ~# (CDouble -> CDouble)))
```


Thus, all that needs to be done is take the desugaring code in `DsForeign.hs` and adapt it to produce the same code, inline, for anonymous FFI imports.


## Future work



Ideally we’d like to be able to call foreign code whose location (be it an address for C or some string for JavaScript) is only known at runtime.  An ad-hoc way to achieve this right now for C is [
libffi](https://sourceware.org/libffi/), which lets us interface with code following the C calling convention programmatically.  However, using such methods from Haskell is always going to incur some overhead.  For instance, the [
current bindings](http://hackage.haskell.org/package/libffi) to libffi for Haskell receive the arguments to the C function to call in an Haskell list.



For this reason, after we implement anonymous FFI imports as described we plan to extend them to be able to accept the location of the code at runtime, as a standard Haskell expression.


