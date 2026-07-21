# Struct Code Generation (`struct`)

## 1. Front-end support for Structs

### Keywords and Grammar
The `struct` keyword is recognized by the lexer and parser, generating AST nodes that handle struct definitions and declarations.
- Parser tokens: `STRUCT`
- Grammar rules: `struct_specifier`, `struct_declaration_list`, `struct_declaration`, `struct_declarator`.

### AST Nodes
- `StructSpecifier`: Represents the struct definition and captures its layout/members.
- `MemberAccess` (`.`): Represents direct access to a struct's member.
- `PointerAccess` (`->`): Represents access to a struct's member via a pointer.

---

## 2. Context and Layout Tracking

The compiler backend tracks struct sizes and memory layouts in the `Context` object.

The relevant functions are used in `src/ast_context.cpp`:
- `DefineStruct`: Registers the total size and blueprint of a struct's members.
- `AddStructMember`: Maps a member name to its specific byte offset within the struct.
- `GetStructSize` / `GetStructMemberOffset`: Retrieves layout information during variable allocation and member access.
- `SetVarStructTag` / `GetVarStructTag`: Associates a specific local variable with its corresponding struct tag so the compiler knows which layout to use.

---

## 3. Struct Declarations and Initialization

Declarations are processed in `src/ast_declaration.cpp` (`EmitInitForDeclarator`).

When a struct variable is declared:
- The compiler checks if the declaration includes a `struct_tag`.
- It paasses the `context.GetStructSize(struct_tag)` function to determine the total byte size required.
- It allocates a local stack slot of that size using `context.AllocateLocalSlot(name, size)`.
- It tags the variable using `context.SetVarStructTag(name, struct_tag)` and assigns it `ValueType::STRUCT`.

---

## 4. Struct Member Access

Implemented in `src/ast_expression.cpp` (`MemberAccess::EmitAddress` and `MemberAccess::EmitRISC`).

Lowering shape for `a.b`:
1. **Base Address**: Evaluates the LHS (`a`) to retrieve its base memory address in `a0`.
2. **Offset Calculation**: Queries the context for the offset of member `b` using the struct tag associated with `a`.
3. **Pointer Arithmetic**: Emits `addi a0, a0, <offset>` to shift the pointer to the exact member.
4. **Load**: If reading the value (via `EmitRISC`), it emits `lw a0, 0(a0)` (or type-specific load) at the resolved address.

For assignments (`a.b = x`), `AssignmentExpr::EmitRISC` calls `EmitAddress` on the `MemberAccess` node, evaluates the RHS, and issues a store instruction (`sw`) to that computed address.

---

## 5. Function ABI Handling for Structs

Implemented in `src/ast_function_definition.cpp` (`FunctionDefinition::EmitRISC`).

When structs are passed by value as function parameters:
- The total byte size is divided by 4 to determine the number of 32-bit words it consumes.
- The compiler iterates through the struct's words, assigning them to available integer registers (`a0` through `a7`).
- If integer registers are exhausted, the remaining words are loaded from the caller's stack frame.
- The words are sequentially spilled into the callee's local frame so the struct is reconstructed contiguously in local memory.
