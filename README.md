# Perceus for OCaml (notes)

## 2022-10-03

- Created repo
  - https://github.com/1ntEgr8/ocaml
- Edited dup/drop assembly to use less instructions
- Discovered error while building system from scratch
  - compat-32 is set during the build which causes our system to crash since we don't support ref-counting for 32-bit systems
  - `Wosize_hd` fails as a result
  - possible fix is to disable compat-32 for now

## 2022-09-18

- Figure out what code Koka generates
  - Looked at `src/refcount.c.s`
- Added a new primitive to read refcount
  - Remember to first run `make coldstart` before `make coreall opt-core`
  - Currently it just returns the entire object header
- dup/drop asm codegen:
  - Mark rax, r10, r11 as destroyed
  - In asmcomp/amd64/emit.mlp, move the argument into rax and emit a call into an assembly routine defined in runtime/amd64.s
  - runtime/amd64.s has two new functions: caml_obj_rc_dup, caml_obj_rc_drop
  - Note: both routines check if the object is a block

## 2022-09-10

- https://github.com/ocaml/ocaml/blob/4.14.0/asmcomp/selectgen.ml
  - instruction selection (cmm to machine-independent assembly)
- status of adding dup/drop instruction:
  - parsetree: done
  - typedtree: done (may need to edit typechecking portion)
  - lambda: done
  - clambda: done
  - flambda: ignored
  - bytecode gen: ignored
  - cmm: done
  - selectgen: done
- some concerns:
  - not sure what effect and coeffect mean in selectgen
    - selected arbitrary effect for dup/drop, will have to stare at it more to see if it breaks things
    - CSEgen has certain operations marked as "handled specially"; couldn't find the code that handles `Ialloc` and `Ipoll` for amd64
- played around with register allocation
  - need to understand how the calling convention is implemented
  - tried this:
    - marked `r10`, `r11` as destroyed in `destroyed_at_oper`
    - used `r10`, `r11` destructively in Idup, Idrop
  - expected:
    - codegen should save `r10`, `r11` on the stack
  - actual:
    - no saving is done?

## 2022-09-06

Plan of attack: 

1. Make it possible to write dup/drop as OCaml functions 

```ocaml
dup : 'a -> () 

drop : 'a -> () 
```

(later variants: like guaranteed pointer drop, drop_ptr : 'a -> ()) 

2. These functions should become real instructions `Idup` / `Idrop`.  Extend the `destroyed_by_oper` function (https://github.com/ocaml/ocaml/blob/trunk/asmcomp/amd64/proc.ml) and implement the emitted assembly. 

3. Write a `map(xs,inc)` example (or inlined sum) with that  

4. Change the assembly generation: 
  a. Ialloc should become a call to "rc_fast_malloc"; implement malloc_fast in assembly   (https://github.com/ocaml/ocaml/blob/4.14.0/runtime/amd64.S) 

  b. That assembly will just save/restore the needed registers and call "malloc" and then put the result in r15.  (this is the inefficient version 😊 

  c. Emit assembly for Idup/Idrop, each calls "rc_checked_dup", "rc_checked_drop".  Checked dup does nothing, checked drop will call "free" if the rc was 0 (saving / restoring registers) 

  d. There are multiple layers:
    (A) the assembly instructions generated for each instruction. (for example, drop: "r0 <- p->refcount; if (r0 <= 0) call checked_drop else p->refcount = r0 – 1". (actually, check first if it is a pointer?) 
    (B) the fast "rc_xxx" assembly routines that use a "all callee save" convention. (like caml_gc does now). And (C) actual C code routines where we need to save the right registers.  

5. Add refcount to object headers in the runtime, and turn off garbage collection.  

Instead of :  

```
+--------+-------+-----+ 
| wosize | color | tag | 
+--------+-------+-----+ 
 63    10 9     8 7   0 
```
 
Use: 

```
+--------+--------+-------+-----+ 
| rc     | wosize | color | tag | 
+--------+--------+-------+-----+ 

 63    31 30    10 9     8 7   0 
```
 
(probably use the profile info of 32 bits) 

 
6. Make this work reliably ; add the edge case, atomics etc. 

7. Write "fast" malloc/free unwrappings and link to mimalloc. 

8. Do some initial perf measurements – this will tell if it can work at all. 

 
Keep calling convention clear: 
- OCaml call convention: all caller save (https://github.com/whitequark/ocaml-llvm-ng/blob/master/doc/abi.md) 
- "Alloc calling convention" (or "Runtime calling convention"): all callee save. (like Ialloc does now, and caml_gc) 
- C calling convention: some registers are callee-save (rbx, rbp, r12-r15) 
- OCaml native code notes: https://gist.github.com/kayceesrk/002822b2b7b11e928789

 
We may reserve more registers for our fast Idup/Idrop/Ialloc by modifying https://github.com/ocaml/ocaml/blob/trunk/asmcomp/amd64/proc.ml 

## 2022/09/05 

- Build without boostrapping 
  - First, before making any change to the compiler, get a stable build 
    - `make world` 
  - Now, add your changes 
  - To build the native compiler (without bootstrapping) 
    - `make coreall opt-core` 
  - Using the native compiler 
    - `./boot/ocamlrun ./ocamlopt -I ./stdlib <prog>` 

 
## 2022/08/25 

- Goal: Emit code to invoke custom C function for allocation 
  - Naively calling custom C function by editing emit.mlp doesn't seem like its going to work out. I need to understand exactly what registers to save across the C-OCaml interface 
  - Here's what I tried (it doesn't compile, segfaults): 
    - Add a file called refcnt.c that has a custom routine that prints something 
    - Update amd64.S and add an assembly stub that invokes this routine (doesn't do calling convention stuff) 
    - Edit amd64/emit.mlp to emit a call to this stub whenever an allocation instruction is found 

- Got something simple to work 
  - Save/restore ALL registers 
  - Make the call 
  - Note: printing something to stdout in the custom routine breaks the build system. It messes with the generation of the `.depend` file. Use stderr :)

## 2022/08/24 

- Compilation pipeline 
  - Parsetree -> Typedtree -> Lambda -> CLambda -> Cmm -> Mach -> code gen 
  - Lose info on the type of block when lowering from CLambda to Cmm 
    - So, we cannot insert dups/drops in Cmm (or it will be a little more complicated)
  - Allocation in a separate heap 
    - https://github.com/ocaml/ocaml/blob/4.14.0/runtime/caml/address_class.h#L16 
      - Classification of addresses 
      - Explains page table, naked pointers 
    - Proposal: 
      - Similar to kk_lib, use malloc/mimalloc to create and manage a separate heap (not managed by GC) 
      - Compile allocation instruction to allocate in this new heap 
      - Compile drop instruction (which needs to be added) to deallocate in this new heap 
- Changing the obj header format 
  - Edit https://github.com/ocaml/ocaml/blob/4.14.0/runtime/caml/mlvalues.h in a bc-way 

## 2022/08/22 

- how are the gc assembly snippets generated 
  - Does it merge allocations? 
    - Yes, https://github.com/ocaml/ocaml/blob/trunk/asmcomp/comballoc.ml 
    - Runs the `comballoc` pass during codegen 
    - We can simply disable this pass. However, disabling it could potentially break some debugging and memprof tests 
  - A special register which saves the allocation pointer is reserved for each architecture 
    - `r15` for amd64 
    - Grepping for "allocation pointer" in the `asmcomp` directory will list the others 
    - Declared in `asmcomp/{architecture}/proc.ml` 

  - https://github.com/ocaml/ocaml/blob/trunk/asmcomp/amd64/emit.mlp#L614-L643 
    - Generation of GC assembly for amd64 
    - `n` is not a constant; depends on the value set in the `Ialloc` instruction 
    - GC is only called at allocation and polling sites 
      - Polling site determined by the `Ipoll` instruction 
  - Calling C from OCaml (links specific to amd64) 
    - PREPARE_FOR_C_CALL + C_call 
      - https://github.com/ocaml/ocaml/blob/trunk/runtime/amd64.S#L372-L385 
    - Stack switching (relevant only for 5.0) 
      - https://github.com/ocaml/ocaml/blob/trunk/runtime/amd64.S#L211-L296 
      - SWITCH_OCAML_TO_C 
      - SWITCH_C_TO_OCAML 
      - SWITCH_OCAML_STACKS 
    - General structure 
      - Save regs (typically alloc ptr and any scratch regs you want to clobber) 
      - Switch to C stack 
      - Make the call 
      - Switch to OCaml stack 
      - Restore regs 
    - Issue: ref counting functions will be invoked frequently, making a C call the naïve way is expensive! 
      - Compile ref counting functions to use a subset of registers, and then only save those regs before making the C call 
      - Inline ref count instructions? 

## 2022/08/10 

- Asm codegen 
  - C-- variant specialized for OCaml 
    - `make_alloc` instruction 
    - [Unboxed objects and Polymorphic Typing, Xavier Leroy (POPL 1992)](https://dl.acm.org/doi/pdf/10.1145/143165.143205)
    - [asmcomp/cmm.mli](https://github.com/ocaml/ocaml/blob/4.14/asmcomp/cmm.mli)
  - `emit_call_gc` at gc call sites 
    - Alloc 
    - Poll sites (after a loop, sys call) 
    - Gc codegen for amd64 
      - https://github.com/ocaml/ocaml/blob/4.14/asmcomp/amd64/emit.mlp#L253-L257 
- Where to add dup/drop instructions? 
  - Seems doable in Lambda form, the code-generator knows where to insert heap allocation calls 

## 2022/08/09 

- Read the "Compiler and Runtime System" section of Real World OCaml 
- Memory representation 
  - Blocks vs values, pointer tagging 
  - Header format 
    - Block size: 22 or 54 bit  
    - Color: 2 bit 
    - Tag: 8 bit 
  - Opts 
    - Float-array 
    - Variants without payload 
  - Defs: [runtime/caml/mlvalues.h](https://github.com/ocaml/ocaml/blob/4.14/runtime/caml/mlvalues.h)
- IR 
  - [Parsetree](https://github.com/ocaml/ocaml/blob/4.14/parsing/parsetree.mli)
  - [Typedtree](https://github.com/ocaml/ocaml/blob/4.14/typing/typedtree.mli)
    - Has typing annotations 
  - [Lambda form](https://github.com/ocaml/ocaml/blob/4.14/lambda/lambda.mli)
    - Untyped
    - Variant: Closed Lambda Form 
    - `makeblock` instruction (and other heap related ops) 

 
## 2022/08/08 

- Built OCaml 4.14.0 
- Looked at [parser/parsetree.mli](https://github.com/ocaml/ocaml/blob/4.14/parsing/parsetree.mli) and [typing/typedtree.mli](https://github.com/ocaml/ocaml/blob/4.14/typing/typedtree.mli) AST definitions 
- Tested making changes to the GC; tested using program shown below 
  - Added print statement to `caml_gc_dispatch` ([runtime/minor_gc.c](https://github.com/ocaml/ocaml/blob/4.14/runtime/minor_gc.c#L461)) 
  - Tried disabling `caml_gc_dispatch` by simply returning 
    - Segfaulted 
  - Tried disabling minor heap GC and major heap GC by commenting out fn calls 
    - Either segfaults or hangs 
  
  - Simple way to disable 
    - Set `Caml_state->requested_minor_gc` and `Caml_state->requested_major_slice` to zero in `caml_gc_dispatch` 
    - https://github.com/ocaml/ocaml/blob/4.14/runtime/minor_gc.c#L483-L484 
- Dump assembly 
  - `ocamlopt –S <file>` 

Test program (lifted from https://ocaml.org/docs/garbage-collection) 

``` 
let rec iterate r x_init i = 
  if i = 1 then x_init 
  else 
    let x = iterate r x_init (i - 1) in 
    r *. x *. (1.0 -. x) 

let () = 
  Random.self_init (); 
  for x = 0 to 100 do 
    let r = 4.0 *. float_of_int x /. 640.0 in 
    for i = 0 to 39 do 
      let x_init = Random.float 1.0 in 
      let x_final = iterate r x_init 500 in 
      let y = int_of_float (x_final *. 480.) in 
      () 
    done 
  done; 

  Gc.print_stat stdout 
``` 
