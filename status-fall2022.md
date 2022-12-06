# Status: fall 2022

## Research Goal

Through this project, we wish to replace OCaml's garbage collector with the Perceus reference counting system.

Along the way, we wish to study the following:

- What fundamental changes need to be made to an industrial strength compiler (like that of OCaml) to retro-actively use Perceus for memory management?
- Did we have to deviate from an ideal implementation owing to the structure of the compiler? If so, what was the impact of any design decisions that were made?
- How does the compiled code perform with Perceus?


## Current status

- Disabled generation of GC code for the x86 code generator
- Implemented emission of several reference counting primitives
    - Switching into OCaml's C runtime has a non-trivial overhead owing to OCaml's callee-save calling convention. We try to eschew switching into the C runtime as much as possible through fast paths, saving as few registers as possible.
    - Memory allocation is done using [mimalloc](https://github.com/microsoft/mimalloc), augmented with special fast path code inlined in the OCaml x86 assembly runtime.
- In the process of automating dup/drop insertion for closures
    - Complicated by the presence of infix headers and mutually recursive closures.

## Next steps

Now that we have most of the reference counting primitives implemented and benchmarked, we will start implementing the Perceus algorithm to automate the refcount operation insertion. Our goal is to get something substantive working by end of January so that we can work on a submission to ICFP'23.

```mermaid
gantt
    dateFormat MM-DD-YYYY
    
    Implement basic Perceus :t1, 12-06-2022, 20d
    Implement Perceus with drop specialization :t2, after t1, 20d
    Evaluate on OCaml benchmarks :t3, after t2, 01-30-2023
    ICFP submision :t4, after t3, 03-01-2023
```
