# Scheme compiler

An educational Scheme compiler written in OCaml. It translates Scheme source into
x86-64 NASM assembly, which the supplied `makefile` assembles and links into a
native Linux executable.

## Compiler behaviour

Compilation proceeds through four stages:

1. **Reader:** parses Scheme values and expressions using the parser combinators
   in `pc.ml`.
2. **Tag parser:** builds an abstract syntax tree and expands forms such as `let`,
   `let*`, `letrec`, `cond`, `and`, and quasiquote into core expressions.
3. **Semantic analysis:** resolves lexical variable addresses, identifies tail
   calls, and boxes variables where needed to share mutation between closures.
4. **Code generation:** emits assembly, constant and global-variable tables,
   closures, and runtime support. Tail calls reuse the current stack frame.

By default, `init.scm` is prepended to each program. Together with the assembly
primitives, it provides arithmetic, comparisons, list/string/vector operations,
and procedures such as `map`, `apply`, and `fold-left`.

The implemented language includes:

- Integers, rational fractions, real numbers, booleans, characters, strings,
  symbols, pairs, lists, vectors, and the literal `#void`.
- `quote`, quasiquote, unquote, and unquote-splicing.
- `if`, `begin`, `and`, `or`, `cond`, `define`, and `set!`.
- Lexical closures, recursion, and lambdas with fixed or rest parameters.
- `let`, `let*`, `letrec`, and leading internal definitions in lambda bodies,
  which are expanded into `letrec`.
- Line comments (`; ...`), expression comments (`#; expression`), and nested paired
  comments (`{ ... }`). Symbol names are converted to lowercase.

At runtime, top-level expressions execute in source order. Each result other
than `#void` is automatically printed on its own line, so definitions and
assignments normally produce no output. Only `#f` is false in conditionals.
Procedure arguments are evaluated from right to left, before the procedure
expression. On normal completion, the runtime prints a memory-usage message.
Runtime checks report errors such as undefined variables, calling a non-procedure,
and incorrect argument counts or types.

## Requirements

- An x86-64 Linux environment; the build uses ELF64 objects and GCC's
  `-m64 -no-pie` options.
- `ocaml`, `nasm`, `gcc`, and GNU `make` available on `PATH`.

Run the following commands from the project directory containing `compiler.ml`.
The compiler loads `pc.ml`, `init.scm`, and the assembly runtime files using
relative paths.

## Compile and run a program

Create a Scheme source file:

```sh
cat > example.scm <<'SCHEME'
(define (square x) (* x x))
(square 6)
(map square '(1 2 3 4))
(+ 1/2 1/3)
SCHEME
```

Compile it to assembly by loading the compiler and calling its OCaml API:

```sh
ocaml -noinit -stdin <<'OCAML'
#use "compiler.ml";;
Code_Generation.compile_scheme_file "example.scm" "example.asm";;
OCAML
```

Successful compilation prints `!!! Compilation finished. Time to assemble!`
and writes `example.asm`. Assemble, link, and run it:

```sh
make example
./example
```

The program prints:

```text
36
(1 4 9 16)
5/6
```

A final `!!! Used ... bytes of dynamically-allocated memory` message follows;
the byte count depends on the program and its runtime allocations.

Use `make <name>` for an existing `<name>.asm` file. The build produces the
executable `<name>` and an assembly listing `<name>.lst`; the intermediate object
file may be removed by Make. Recompile the Scheme source before rebuilding after
source changes: the Makefile only handles assembly and linking.

## Use the compiler interactively

Start `ocaml` from the project directory, then enter:

```ocaml
#use "compiler.ml";;

(* Input Scheme file, then output assembly file. *)
Code_Generation.compile_scheme_file "example.scm" "example.asm";;

(* Output assembly file, then Scheme source string. *)
Code_Generation.compile_scheme_string "sum.asm" "(+ 1 2 3)";;

#quit;;
```

Back in the shell, run `make sum` followed by `./sum` to execute the string
example. Both compile functions write assembly only and overwrite the specified
output file. There is no command-line argument parser: running
`ocaml compiler.ml` alone only loads the compiler definitions.

For experiments with only the assembly primitives, call
`Code_Generation.do_not_use_init_file ();;` before compiling. Restore the default
with `Code_Generation.use_init_file ();;`. Disabling initialization removes
Scheme-defined procedures such as `+` and `map`.

## Current limitations

- `compile_and_run_scheme_string` invokes `make -f testing/makefile`, and the
  `test`/`test2` aliases also use paths under `testing/`. That directory is absent
  from this checkout; use the compile-and-build commands above.
- The reader recognizes string interpolation such as `"value: ~{(+ 1 2)}"`, but
  expands it into calls to `format`. The bundled library does not define
  `format`, so interpolation requires a suitable user-provided definition.
- The compilation entry points do not check that the reader consumed the whole
  source. An unreadable suffix can be silently ignored, so a compilation-success
  message does not guarantee that every character was parsed.
- The runtime uses a fixed 1 GiB allocation area with no garbage collection or
  allocation-bound checks. It is intended for small educational programs.

## Project files

| File | Purpose |
| --- | --- |
| `compiler.ml` | Reader, tag parser, semantic analysis, code generator, and compilation API. |
| `pc.ml` | Parser-combinator library used by the reader. |
| `init.scm` | Scheme library included by default in compiled programs. |
| `prologue-1.asm` | Runtime type tags, object-layout macros, and data-section setup. |
| `prologue-2.asm` | Executable entry point and initial stack setup. |
| `epilogue.asm` | Primitive procedures, printing, allocation, and runtime error handlers. |
| `makefile` | Assembly and linking rules, plus optional listing/disassembly PDF targets. |
