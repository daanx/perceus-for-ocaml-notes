# OCaml Closures

## Memory representation (64-bit)

```
memory representation of OCaml closures

               +------------+------------+------------+
               |   header   | code value | info field |
               +------------+------------+------------+
size in bits         64           64           64
```

### `header`

OCaml header with tag = 247

### `code value`

Pointer to the code segment of the closure

### `info field`

Metadata of the closure.

It is structured as follows:
```
info field of an OCaml closure

         +---------+-------------------+---+
         |  arity  |      field no     | 1 |
         +---------+-------------------+---+
bits      63     56 55                2 1  0
```
- `arity`: Arity of the closure (signed)
- `field no`: Field number of the first word of the environment, i.e., offset (in words) from the closure to the environment

Eg:

`info field` = '0x100000000000005' means:
- The closure takes one argument
- The closure environment is at a +2 word offset from the closure 

## Representation in Lambda form

## Representation in CLambda form

## Representation in Cmm form

## Inserting dups/drops for closures
