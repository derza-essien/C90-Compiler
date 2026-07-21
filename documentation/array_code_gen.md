# Array Code Generation ( `[]` )

## 1. Front-end support for Arrays

### Grammar and AST Nodes
Arrays are recognized via postfix bracket notation `[]` in the parser.
- `ArrDeclarator`: Captures array declarations, including an optional `constant_expression` for size.
- `ArrAccess`: Captures array indexing expressions (`a[i]`).
- `InitializerList`: Captures bracketed initialization expressions like `{1, 2, 3}`.

---

## 2. Array Declarations (Global vs Local)

Array sizing and allocation happen in `src/ast_declaration.cpp` through the  `EmitInitForDeclarator` function.

**Size Calculation**:
- Base element size is determined (e.g., 4 for `int`, 1 for `char`).
- This is multiplied by the evaluated size of the `ArrDeclarator`.

**Local Scope**:
- The total size is passed to `context.AllocateLocalSlot` to reserve stack space.
- Initializer lists evaluate their items and emit sequential `sw` instructions to populate the stack memory.

**Global Scope**:
- If `context.IsGlobalScope()` is true, the array bypasses stack allocation.
- The compiler emits `.data`, `.globl`, and `.size` directives.
- If an `InitializerList` is present, it is unrolled into sequential `.word` (or `.byte`) directives in the assembly file.
- If uninitialized, it emits `.zero <size>`.

---

## 3. Array to Pointer Decay

This is implemented in `src/ast_identifier.cpp` through the `Identifier::EmitRISC` function.

When an array identifier is used in an expression, it must decay into a pointer to its first element rather than loading the entire array structure.
- The compiler compares the allocated `GetVarSize(name)` against the single-element size `GetSizeOfValueType`.
- **Local Arrays**: If it's a local array, it emits `addi a0, s0, <offset>` to materialize the stack address into `a0`.
- **Global Arrays**: If it's a global array, it emits `lui a0, %hi(name)` and `addi a0, a0, %lo(name)` to materialize the global data address.

---

## 4. Array Indexing and Pointer Arithmetic

Implemented in `src/ast_expression.cpp` (`ArrAccess::EmitAddress` and `ArrAccess::EmitRISC`).

Lowering shape for `a[i]`:
1. **Base evaluation**: Evaluate LHS (`a`) and leave the base memory address in `a0`.
2. **Save base**: Save `a0` to the stack (`sw a0, 12(sp)`).
3. **Index evaluation**: Evaluate RHS (`i`) into `a0`.
4. **Scale index**: Multiply the index by the element size (`GetElementSize`). It uses `slli` for powers of 2 (e.g., `slli a0, a0, 2` for ints) or `mul` for arbitrary sizes (like structs).
5. **Addition**: Reload the base address into `t0` and add the scaled index (`add a0, t0, a0`).
6. **Load**: If doing a read, issue a load instruction depending on element size (e.g., `lbu` for 1-byte elements, `lw` for 4-byte).
