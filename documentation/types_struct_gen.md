# Typedef Code Generation (`typedef`)

## 1. Front-end support for Typedefs

### Grammar
The `typedef` keyword modifies a declaration to create a type name instead of a variable.
- Uses the `TYPEDEF` storage class specifier.
- When defined, future instances are matched by the lexer/parser as `TYPE_NAME` tokens instead of standard identifiers.

---

## 2. Symbol Table Tracking

Implemented in `src/ast_typedefs.cpp` and `src/ast_typedefs.hpp`.

Instead of generating assembly, `typedef` acts purely as a semantic front-end construct.
- The compiler maintains a scoped symbol table `std::vector<std::unordered_map<std::string, TypedefInfo>>`.
- `TypedefInfo` tracks three core properties:
  1. `TypeKind` (i.e. the data kind of data type e.g., `INT`, `STRUCT`)
  2. `struct_tag` (if it aliases a struct)
  3. `pointer_depth` (tracks if the typedef inherently includes pointer stars).

---

## 3. Typedef Registration

When a `Declaration` node evaluates and sees its `StorageClass` is `TYPEDEF`, it skips normal code generation.
- Instead, it calls `ast::RegisterTypedefNames`.
- This function walks through the declarator list, calculates the `pointer_depth` (via `LocalGetPointerDepth`), and registers the alias name into the current scope's map.

---

## 4. Type Resolution and Merging

When the parser later encounters a `TYPE_NAME` being used to declare a new variable:
- It creates a base `DeclSpec` using `GetTypedefType` to fetch the underlying `TypeKind`.
- It restores the underlying pointer properties using `ds->SetPointerDepth(GetTypedefPointerDepth($1))`.
- If `TYPE_NAME` is mixed with other specifiers, `MergeTypeNameWithDeclSpec` handles cascading the saved base type, struct tag, and pointer depth to the new declaration.

By restoring the underlying pointer depth and base type, the rest of the backend (like `ast_declaration.cpp` and `ast_expression.cpp`) handles the variable exactly as if the original long-form type was written out.
