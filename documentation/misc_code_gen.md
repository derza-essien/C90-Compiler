# Miscellanoeus Code Generation

The folder named `misc` includes tests involving enums, switches, and types.

**The readme for the types has been done separately in [types.md](https://github.com/derza-essien/C90-Compiler/new/main/documentation/types.md).**

## Enum Code Generation (`enum`)


### 1. Front-end support for Enums

**Grammar and AST Nodes:**

The `enum` keyword is recognized by the parser to define enumerated types and constants.
- **Parser tokens**: `ENUM`.
- **Grammar rules**: `enum_specifier`, `enumerator_list`, `enumerator`.
- **AST Nodes**: 
  - `EnumSpecifier`: Captures the definition of the enum, including its optional tag and list of enumerators.
  - `Enumerator`: Represents individual constants inside the enum block (`IDENTIFIER` or `IDENTIFIER = constant_expression`).

---

### 2. Context and Value Tracking

Enums in C are functionally treated as integer constants. The compiler backend stores these mappings directly in the `Context` object so they can be substituted at compile-time.

This is implemented in `src/ast_context.cpp`:
- `DefineEnum` / `RegisterEnumConstant`: Maps a string identifier to its computed integer value and stores it in the `enum_values_` map.
- `TryGetEnum` / `TryGetEnumConstant`: Checks if a given string matches a registered enum constant and retrieves its value.

---

### 3. Usage and Identifier Resolution

As enum constants do not exist in memory, they must be injected as immediate values during code generation.

Implemented in `src/ast_identifier.cpp` (`Identifier::EmitRISC`):
- When an `Identifier` node is evaluated, the compiler first checks if it is an enum constant using `context.TryGetEnum(identifier_, enum_val)`.
- **If True**: It completely bypasses the variable lookup process and directly emits a load immediate instruction: `li a0, <enum_val>`.
- **If False**: It falls back to searching for a local or global variable offset.

## Switch Statement Code Generation (`switch`)

### 1. Front-end support for Switch

**Grammar and AST Nodes:**

The parser recognizes `switch`, `case`, and `default` control structures.
- **Parser tokens**: `SWITCH`, `CASE`, `DEFAULT`.
- **Grammar rules**: `selection_statement`, `labeled_statement`.
- **AST Nodes**: `SwitchStatemnt`, `CaseStatement`, `DefaultStatement`.

---

### 2. Pre-pass: Case Collection

Switch statements require knowing all case values and their jump labels before emitting the switch body. This is handled by a recursive pre-pass.

Implemented in `src/ast_statement.cpp` (`CollectCases`):
- The `SwitchStatemnt::EmitRISC` function initializes an empty list of cases and a default label (which defaults to the switch's end label if no `default` is provided).
- It calls `stmt_->CollectCases()` on its body, which traverses down into compound statements.
- When a `CaseStatement` is found, it evaluates its constant expression (either an `IntConstant` or an `Identifier` resolving to an enum), generates a unique label (`.Lcase_label_N`), maps it in the context using `context.SetSwitchLabel()`, and pushes the `[value, label]` pair to the case list.
- When a `DefaultStatement` is found, it generates a unique `.Ldefault_label_N` and updates the default target.

---

### 3. Switch Emission and Branching

Implemented in `src/ast_statement.cpp` (`SwitchStatemnt::EmitRISC`).

Lowering shape for `switch(expr)`:
1. **End Label & Break**: A unique `end_label` is generated and pushed to the context (`context.PushBreakLabel(end_label)`) so any `break;` statements inside the switch know where to jump.
2. **Evaluate Condition**: The switch `expr_` is evaluated, leaving the test value in `a0`.
3. **Emit Jump Table / Branches**: The compiler loops through the collected `cases` list. For each case, it emits:
   - `li t1, <case_value>`
   - `beq a0, t1, <case_label>`.
4. **Emit Default Fallback**: After checking all specific cases, it emits an unconditional jump to the default label: `j <default_label>`.
5. **Emit Body**: The switch body (`stmt_`) is finally compiled. As the compiler encounters the actual `CaseStatement` and `DefaultStatement` nodes inside the body, they fetch their pre-assigned labels via `context.GetSwitchLabel(this)` and emit them as assembly jump targets (e.g., `.Lcase_label_0:\n`).
6. **Cleanup**: The `end_label` is placed at the very bottom, and the break label is popped from the context.# Switch Statement Code Generation

## 1. Front-end support for Switch

### Grammar and AST Nodes
The parser recognizes `switch`, `case`, and `default` control structures.
- **Parser tokens**: `SWITCH`, `CASE`, `DEFAULT`.
- **Grammar rules**: `selection_statement`, `labeled_statement`.
- **AST Nodes**: `SwitchStatemnt`, `CaseStatement`, `DefaultStatement`.

---

## 2. Pre-pass: Case Collection

Switch statements require knowing all case values and their jump labels *before* emitting the switch body. This is handled by a recursive pre-pass.

Implemented in `src/ast_statement.cpp` (`CollectCases`):
- The `SwitchStatemnt::EmitRISC` function initializes an empty list of cases and a default label (which defaults to the switch's end label if no `default` is provided).
- It calls `stmt_->CollectCases()` on its body, which traverses down into compound statements.
- When a `CaseStatement` is found, it evaluates its constant expression (either an `IntConstant` or an `Identifier` resolving to an enum), generates a unique label (`.Lcase_label_N`), maps it in the context using `context.SetSwitchLabel()`, and pushes the `[value, label]` pair to the case list.
- When a `DefaultStatement` is found, it generates a unique `.Ldefault_label_N` and updates the default target.

---

## 3. Switch Emission and Branching

Implemented in `src/ast_statement.cpp` (`SwitchStatemnt::EmitRISC`).

Lowering shape for `switch(expr)`:
1. **End Label & Break**: A unique `end_label` is generated and pushed to the context (`context.PushBreakLabel(end_label)`) so any `break;` statements inside the switch know where to jump.
2. **Evaluate Condition**: The switch `expr_` is evaluated, leaving the test value in `a0`.
3. **Emit Jump Table / Branches**: The compiler loops through the collected `cases` list. For each case, it emits:
   - `li t1, <case_value>`
   - `beq a0, t1, <case_label>`.
4. **Emit Default Fallback**: After checking all specific cases, it emits an unconditional jump to the default label: `j <default_label>`.
5. **Emit Body**: The switch body (`stmt_`) is finally compiled. As the compiler encounters the actual `CaseStatement` and `DefaultStatement` nodes inside the body, they fetch their pre-assigned labels via `context.GetSwitchLabel(this)` and emit them as assembly jump targets (e.g., `.Lcase_label_0:\n`).
6. **Cleanup**: The `end_label` is placed at the very bottom, and the break label is popped from the context.
