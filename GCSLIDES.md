## Code

From [code](https://justine.lol/sectorlisp2/#listing).

```
function Copy(x, m, k) {
  return x < m ? Cons(Copy(Car(x), m, k),
                      Copy(Cdr(x), m, k)) + k : x;
}

function Gc(A, x) {
  var C, B = cx;
  x = Copy(x, A, A - B), C = cx;
  while (C < B) Set(--A, Get(--B));
  return cx = A, x;
}
```
Unwound to:
```
function Copy(x, m, k) {
  if (x >= m) {
  	return x;
  } else {
    return k + Cons(Copy(Car(x), m, k),
                      Copy(Cdr(x), m, k));
  }
}

function Gc(A, x) {
  var C;
  var B = cx;
  var x = Copy(x, A, A - B)
  C = cx;
  while (C < B) Set(--A, Get(--B));
  cx = A;
  return x;
}
```
## Salient Points

- there are only 2 memory spaces (1) atoms, (2) lists
- pure functions -> strict stack allocation
- Copy/move must be atomic - creates temporary inconsistencies
- move overwrites unused cells on stack
- no circular (cyclic) lists
- Eval() calls GC() every time - cleans up after itself

## Memory Spaces

- 256 bytes (128 cells)
- address is 1 byte (-127 .. +128)
- traditional Lisp divides all memory into cells `{car, cdr}`
- Sector Lisp divides all cells into 2 types: List, Atom
- Sector Lisp uses sign as type
	- `>0` Atom
	- `<0` List
	- `=0` NIL

	- (ammenable to assembler - branch on flags - BG, BL, BZ)

N.B. 1-byte address *could* address 256 cells (512 bytes).

## Copy

- Follow pointers of cells to keep.
- Copy cells to keep using CONS() + offset.
- Start at returned value, copy all cells that belong to the returned value
- Strict stack allocation -> offset is constant over one GC cycle

## Copy/Move is Atomic

- creates cells containing pointer offsets in CDRs
- temporarily inconsistent
- offsets point to new locations of cells
- compacts memory, all cells on stack are in-use
- SP (stack pointer) adjusted to point to next free stack location

`function Copy(x, m, k) { ...`

- `x` is the list to keep (non garbage)
- `k` is offset to final location
- `m` is a barrier
	- the previous SP (stack pointer)
	- all cells >= m remain untouched (this includes atoms in +ve space)
- Copy calls CONS to allocate new cells with offset, on the stack
- Copy adds k to address returned by CONS
- Copy does not copy atoms, they are simply returned verbatim.
- Likewise, Copy does not copy cells which were allocated previously.

## Move Overwrites Unused Cells on Stack

- `while (C<B)...`
- offsets become "correct" after move

## Pure Functions

- strict stack allocation
- args always evaled before function is called
- args -> results put on stack, always allocated and GCed before function is called
- arg is always one of (2) List, or, (1) Atom
- results in smaller size (no edge cases)
- no heap, just stack

## Offset
- `k`
- applied to address of all new CONS cells 
	- during Copy, it is known that a copied cell will be relocated during compaction
- CAR is either (1) an atom, (1a) NIL, or (2) a CONS cell 
	1. ATOM is always +ve
	0. NIL is always 00
	2. CONS cell always comes from evaled arg
		- -> arg has already been EVALed and GCed
		- CONS cell, pre-EVALed as arg, is always more +ve than `m` 
			- (or `= m`)
			- `>= m`
	
	## No Cyclic Lists
	
	- -> less code
	- pure functions *cannot* create cycles
		- pure functions cannot refer to "future" data, or self
		- `set!` is required to create cycles
		- `set!` -> epicycles + accidental complexity
		- `set!` required by heap access (aka random access)
	- `set!` is `MOV`/`STO` low-level assembly function
	- corollary: Pure Functional notation does not describe `set!`/`heap` well 
		- force-fitting `set!` into FP leads to more code (aka "code bloat")
		
	## Visuals
	[animation (silent, quick)](https://youtu.be/gn5E1jyzqro)
	[animation (silent, real time)](https://www.youtube.com/watch?v=TF0FzcBkV60)
	
	## Optimization
	
	## Peel Instead of TCO
	
	Many functions pass a parameter verbatim to a callee.
	
	In `Eval (expr, env)`, the `env` alist is searched linearly for the first occurrence of a name.
	
	Shortening the length of the env alist is one way to speed up the interpreter.
	
	Optimization: if a variable is unaltered from call to call, then don't add it to the env alist (previous values, already on the alist, hold the same value), and, don't re-eval the variable (since we already have its value).
	
	The [Peel function](https://justine.lol/sectorlisp2/#listing) suggests this kind of simple optimization.
	
	TCO (Tail Call Optimization) accomplishes this same optimization, but, with more complexity.  TCO does the job "more thoroughly", than Peel(), but it might not be worth the extra effort in many cases.
	
	## Pure Functional GC
	
	Justine Tunney's GC, in Sector Lisp, demonstrates that GC can be very simple in a Pure Functional paradigm.
	
	When consideration for mutation is added, e.g. in the guise of *heaps* and *random access*, etc., code complexity - code bloat - increases.'
	
	From my perspective, Sector Lisp's main feature is not smallness, but purity of notation.
	
	When we add problems, like forever-looping and concurrency, to the otherwise pure notation, we create code bloat and epicycles.
	
	## Edits
	
	Edited Feb 17, 2022.  Added unwound C code.  Added section Pure Functional GC.