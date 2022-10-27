# OCaml Closures

For an overview of the OCaml closure memory representation, see [this excellent summary](https://github.com/Gbury/ocaml-memgraph/blob/0cc45a10ed488cb9e63d848f61d7c1b7157db1c8/doc/closures.md) by [Gbury](https://github.com/Gbury).

## Inserting dups/drops for closures

### Uncurried closures

Suppose the free variables captured by the closure are `y0, y1, ..., ym`.

Then,
```ocaml
let closure a0 a1 ... an env = <closure body>
```
transforms to
```ocaml
let closure a0 a1 ... an env =
    (* Dup the environment *)
    let x0 = dup (field 0 (offset env)) in
    let x1 = dup (field 1 (offset env)) in 
    ...
    let xm = dup (field m (offset env)) in
    (* Drop the closure *)
    drop env ;
    (* Execute the closure *)
    <closure body>[x0/y0][x1/y1]...[xm/ym]
```
where
- `env : closure_block_addr` is the field implicitly passed by the OCaml compiler that points to the start of the closure block in memory
- `offset : closure_block_addr -> env_addr` returns the address of the start of the closure environment given the start of the closure block.

### Curried closures

TODO
