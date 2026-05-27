# Blue BEX Specification 1.0

> **Scope.** This document defines Blue BEX: a deterministic expression and statement language encoded as Blue-compatible data. It specifies the BEX program model, compilation rules, value model, expression and statement semantics, runtime context, pointer behavior, result accumulation, gas accounting, host output boundary, fixtures, and conformance expectations. It does **not** redefine the Blue node model, BlueId, type resolution, canonicalization, provider behavior, or contract execution. Those belong to the Blue Language Specification and, where applicable, the Blue Contracts and Processor Specification.

Where this document references Blue nodes, BlueIds, Blue documents, Blue reserved fields, schema, or type matching, the Blue Language Specification is the substrate. BEX programs are authored as Blue data and may produce Blue-compatible values, but BEX execution is a computation layer above Blue content.

## Conventions

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**, **MAY**, and **OPTIONAL** are to be interpreted as normative requirement levels.

Sections marked **normative** define required behavior for the relevant conformance profile. Sections marked **informative** explain intent, examples, or implementation guidance.

A requirement is **profile-relative**. A compiler-only implementation is not required to execute programs. An execution implementation is not required to provide every possible host storage backend. A fixture-conformant implementation is required to implement the deterministic fixture model, including exact gas behavior.

---

## 0. Overview

Blue BEX, short for **Blue Expression Objects**, is a deterministic scripting language written as Blue-compatible object trees.

A BEX program is a Blue document or Blue node containing either:

- a root expression under `expr`;
- a statement sequence under `do`;
- an `entry` function name;
- reusable `constants` and `functions`.

BEX has no direct external side effects. It does not mutate the input document, send network requests, perform I/O, execute JavaScript, or execute Blue contracts. Host capabilities are available only through explicitly registered `$intrinsic` processors keyed by BlueId (§6.11); unsupported intrinsic BlueIds fail closed. BEX computes a **BEX Execution Result** containing:

| Result component | Meaning |
|---|---|
| `value` | The returned BEX value, usually converted to a Blue node by the host. |
| `changeset` | Ordered document patch entries accumulated by statement helpers. |
| `events` | Ordered event values accumulated by statement helpers. |
| `gasUsed` | Deterministic gas consumed during execution. |
| `metrics` | Deterministic counters such as expression, statement, function, document-read, and patch counts. |

A typical BEX pipeline is:

```text
BEX Source Blue Document
   -- parse / freeze       --> immutable Blue node view
   -- compile              --> compiled BEX program
   -- execute with context --> BEX Execution Result
   -- host policy          --> store value, apply patches, emit events, or reject
```

BEX is intentionally smaller than a general programming language:

- programs are data, not text source code;
- operators are ordinary Blue object shapes;
- execution is deterministic for fixed inputs;
- functions are statically named and non-recursive;
- local variables are frame slots, not mutable Blue content;
- collection query expressions are deterministic over lists and sorted object keys;
- changes and events are accumulated as data for the host to interpret;
- host extension points are explicit `$intrinsic` calls resolved by BlueId and processor registry.

### 0.1 Operator shape

A BEX expression operator is an object with **exactly one field** whose key begins with `$`.

```yaml
$add:
  - 1
  - 2
```

An object that has more than one field is a literal or computed object, even if one or more field names begin with `$`.

```yaml
# Literal object, not an operator
$foo: 1
bar: 2
```

A statement is stricter. A statement object MUST contain exactly one statement operator field and MUST NOT contain siblings.

### 0.2 BEX and Blue

BEX source is Blue-compatible data, but BEX operators are not Blue Language operators. BEX execution may read Blue documents, Blue type-matching may be used by `$is`, and BEX output may be converted to Blue nodes, but BEX itself does not change BlueId rules.

A BEX implementation MUST preserve the Blue Language boundary:

- BEX expressions MUST NOT appear inside static Blue type-definition fields such as `type`, `itemType`, `keyType`, `valueType`, `blue`, or `schema`.
- Computed output that claims to be Blue content MUST obey Blue node-shape rules at the output boundary.
- A BEX program MUST NOT rely on hash-order, map-order, provider availability, or host side effects for its semantics.

---

## 1. Scope, Goals, Versioning, and Conformance Profiles

### 1.1 Goal

Blue BEX defines a deterministic computation layer for Blue systems with:

- expression evaluation over Blue-compatible values;
- statement execution with local variables, loops, conditionals, returns, and failures;
- typed and structural function arguments through Blue type matching;
- deterministic collection projection, selection, and aggregation expressions;
- deterministic changeset and event accumulation;
- deterministic gas accounting;
- host-controlled document, binding, event, step, and contract reads;
- explicit registered intrinsic dispatch by BlueId for host capabilities;
- conversion of BEX values to Blue nodes under strict Blue output rules.

### 1.2 Out of scope

The following are not defined by this specification:

- Blue Language parsing, BlueId, canonicalization, type resolution, and provider protocols;
- runtime execution of Blue contracts;
- host authorization;
- persistence of patches or events;
- network access, I/O, clocks, randomness, or non-deterministic host calls except through separately specified deterministic intrinsic processors;
- user-interface semantics;
- concurrent execution semantics.

BEX produces data. The host decides whether to apply a patch, emit an event, persist a returned node, or reject a result.

### 1.3 Versioning

This document defines **Blue BEX 1.0**.

A BEX implementation MUST declare the BEX version and conformance profiles it supports. BEX 1.x revisions SHOULD preserve the meaning, result values, diagnostics class, and gas behavior of valid BEX 1.0 fixtures unless explicitly marked as compatible clarifications.

A change that modifies operator recognition, value equality, pointer resolution, patch accumulation, or gas accounting for valid BEX 1.0 programs is an incompatible change and requires a new major BEX version or an explicit compatibility profile.

### 1.4 Conformance profiles

A conforming implementation MUST declare one or more profiles.

#### BEX Compiler Profile

A **BEX Compiler Profile** implementation MUST implement:

- parsing or accepting an immutable Blue node source;
- operator recognition;
- literal and computed Blue object compilation;
- static validation of constants, functions, arguments, statement shapes, operators, reserved user names, and static patterns;
- rejection of BEX expressions in static Blue definition fields;
- function call graph validation, including recursion rejection;
- deterministic compiled-program construction.

A compiler-only implementation is not required to execute a compiled program.

#### BEX Execution Profile

A **BEX Execution Profile** implementation MUST implement the BEX Compiler Profile and additionally:

- BEX value model;
- expression evaluation;
- statement execution;
- runtime context reads;
- pointer resolution;
- local variables and function frames;
- changeset and event accumulators;
- result overlay and `$resultValue`;
- gas metering and exhaustion;
- registered intrinsic invocation and fail-closed unsupported intrinsic behavior;
- runtime errors.

#### BEX Host Boundary Profile

A **BEX Host Boundary Profile** implementation MUST implement the BEX Execution Profile and additionally:

- conversion of BEX values to Blue nodes;
- immutable node snapshots or equivalent trust boundaries;
- document-view semantics for canonical and resolved reads;
- host bindings, event, steps, and current-contract values;
- deterministic simple-value conversion for tests and host APIs.

#### BEX Fixture Profile

A **BEX Fixture Profile** implementation MUST implement the BEX Host Boundary Profile and additionally support the rich fixture format in §16, including exact gas assertions and error-class assertions.

### 1.5 Dependency on Blue Language

A conforming BEX implementation MUST be explicit about the Blue Language version used for:

- Blue node parsing;
- type matching;
- Blue output conversion;
- BlueId computation, when a host exposes node BlueIds to BEX values.

Portable BEX 1.0 programs SHOULD target Blue Language 1.0 and SHOULD avoid host-local Blue features unless a compatibility profile is declared.

---

## 2. BEX Source Documents and Program Model

### 2.1 BEX source as Blue data (normative)

A BEX source is a Blue-compatible node. BEX does not use a separate textual grammar. YAML and JSON authoring follow the host's Blue parser rules.

The root program node MAY contain:

| Field | Shape | Meaning |
|---|---|---|
| `constants` | plain object | Named compile-time constants. |
| `functions` | plain object | Named user-defined functions. |
| `expr` | any BEX expression | Root expression program. |
| `do` | list of statements | Root statement program. |
| `entry` | string | Name of the function to invoke as the root program. |

Other ordinary fields are not program-control fields unless the host profile gives them meaning. Programs commonly include Blue metadata such as `name` or `type`, but those fields do not by themselves affect BEX execution unless read or returned as data.

### 2.2 Program selection (normative)

The root executable is selected as follows:

1. If an explicit host API entry name is supplied, that function is the root executable.
2. Otherwise, if the program node contains an `entry` field, its string value is the root executable name.
3. Otherwise, if the program node contains `expr`, the root executable is that expression.
4. Otherwise, the root executable is the statement list under `do`.
5. If no executable content is present, the root statement program is empty and returns the default result value described in §7.9.

An entry function MUST exist and MUST declare no arguments. If an entry function declares arguments, compilation MUST fail.

### 2.3 Definition node plus program node (normative)

An implementation MAY accept a separate **definition node** and **program node**. When both are supplied:

- constants from the definition node are loaded first;
- constants from the program node are loaded second and replace same-named definition constants;
- functions from the definition node are loaded first;
- functions from the program node are loaded second and replace same-named definition functions.

Replacement is by name. Implementations MUST make replacement deterministic.

### 2.4 Plain name containers (normative)

The following BEX containers are **plain name containers**:

- `constants`;
- `functions`;
- function `args`;
- `$call.args`.

A plain name container MUST be an ordinary object map and MUST NOT use Blue language keys, BEX list-control keys, or Blue wrapper keys as user-defined names.

The reserved user-name set is:

```text
name, description,
type, itemType, keyType, valueType,
value, items,
blueId, blue,
schema, constraints, mergePolicy,
properties, contracts,
$previous, $pos, $replace, $empty
```

A plain name container MUST NOT be a scalar node, list node, pure reference, Blue wrapper node, or list-control node.

### 2.5 Static Blue definition fields (normative)

BEX expressions MUST NOT appear inside static Blue definition fields:

```text
type, itemType, keyType, valueType, blue, schema
```

This rejection applies recursively inside those fields. For example, a computed `type`, a computed `blueId` inside `type`, or a BEX operator embedded in `schema` MUST be rejected at compile time.

The same static rule applies to `$intrinsic.type`: the intrinsic operation BlueId MUST be knowable at compile time.

The purpose is to keep Blue type definitions, schema declarations, and intrinsic operation identity static Blue content, rather than executable BEX content.

### 2.6 Literal escape (normative)

`$literal` returns its body as literal data and prevents nested BEX operator interpretation, except that the compiler MUST still reject BEX expressions in static Blue definition fields before accepting the literal.

Example:

```yaml
$literal:
  $unknownOperator: kept as data
```

Invalid because the nested expression is inside a static Blue type field:

```yaml
$literal:
  type:
    $const: SomeType
```

### 2.7 Operator recognition (normative)

An expression object is a BEX operator if and only if it has exactly one field and that field name begins with `$`.

- If the sole operator name is unknown, compilation MUST fail.
- If an object has more than one field, it is not an expression operator solely by virtue of dollar-prefixed fields.
- Lists, scalars, and objects without an operator shape are literals unless they contain nested BEX expressions in ordinary expression positions.

A statement object MUST have exactly one statement operator field. Unknown statement operators or sibling fields MUST fail compilation.

Null or empty statement items MUST fail compilation.

---

## 3. BEX Values

### 3.1 Value kinds (normative)

BEX has the following runtime value kinds:

| Kind | Meaning |
|---|---|
| `undefined` | Internal absence. Not a Blue value and not valid output. |
| `null` | Explicit empty/null-like value, converted to an empty Blue node at the Blue output boundary. |
| scalar | Text, integer, decimal number, or boolean scalar. |
| object | String-keyed map of BEX values. |
| list | Ordered sequence of BEX values. |
| node cursor | Host-backed Blue node view. |
| frozen node | Immutable Blue node snapshot. |
| overlay | Runtime value representing patched or updated data without mutating its base. |

`undefined` and `null` are distinct. `$exists` is false only for `undefined`; it is true for `null`, `false`, empty text, empty objects, and empty lists.

### 3.2 Undefined (normative)

`undefined` represents absence.

- Reading a missing object key returns `undefined`.
- Reading an out-of-range list index returns `undefined`.
- `$pointerGet` returns `undefined` for a missing path unless a default is supplied.
- Object construction omits fields whose evaluated value is `undefined`.
- Lists MUST NOT contain `undefined` items.
- Converting a root `undefined` value to a Blue node MUST fail.

### 3.3 Null (normative)

`null` is a value, not absence.

- `null` is falsy.
- `null` has size `0`.
- `null` converts to an empty Blue node at the Blue output boundary.
- `null` is distinct from `undefined` for `$exists` and equality.

### 3.4 Scalars (normative)

BEX scalar values are:

- text strings;
- arbitrary-precision integers;
- arbitrary-precision decimal numbers for numeric conversion and comparison;
- booleans.

Floating-point host inputs MUST be finite. `NaN`, `Infinity`, and `-Infinity` MUST be rejected.

Integer conversion is exact. A decimal number can convert to integer only when it has no fractional part. A text string can convert to integer only when it matches canonical decimal integer syntax.

### 3.4.1 Numeric boundary with Blue (normative)

BEX has its own execution numeric model.

BEX integers are arbitrary-precision mathematical integers. BEX decimal numbers are exact decimal execution values for conversion and comparison.

Blue `Integer` values read into BEX become BEX integers exactly. Blue `Double` values read into BEX come from the Blue Language finite IEEE 754 binary64 model. `NaN`, `Infinity`, and `-Infinity` remain invalid at the boundary.

A Blue `Double` read into BEX MUST NOT depend on the original source token spelling. It MUST be derived from the Blue scalar value after Blue parsing and inference. Conversion to the BEX decimal execution value MUST be deterministic and declared by the implementation profile. Under the BEX 1.0 Fixture Profile, the BEX decimal value is produced from the shortest round-tripping decimal rendering of the finite binary64 value.

BEX numeric equality is computation equality, not Blue identity equality. Therefore BEX may treat integer `1` and decimal `1.0` as equal, while Blue Language identity still distinguishes `Integer 1` from `Double 1.0`.

Output conversion back to Blue MUST choose or preserve a Blue scalar type deterministically:

- an exact BEX integer becomes Blue `Integer`;
- a non-integer BEX decimal becomes Blue `Double` only through deterministic finite binary64 conversion, or conversion fails if the value cannot be represented under the declared host profile;
- explicit Blue-shaped output with `type` and `value` MUST obey the Blue Language scalar rules.

When BEX decimal output is converted to Blue `Double`, rounding is permitted only by a deterministic round-to-nearest, ties-to-even binary64 conversion. Conversion MUST fail on overflow or any result that would be non-finite.

BEX equality MUST NOT be used as a substitute for BlueId or Blue scalar identity comparison. The Blue Language numeric model distinguishes integer token inference and Double token inference; Double values are finite IEEE 754 binary64 values, and large Integers require exact canonical decimal representation.

### 3.5 Objects (normative)

BEX objects are maps from text keys to BEX values.

- Key lookup of a missing key returns `undefined`.
- Object key exposure order is deterministic. Implementations MUST expose object keys in lexicographic order over Unicode code points of the unescaped key text for `$keys`, `$entries`, `$forEach` over objects, deterministic simple conversion, and deterministic diagnostics.
- Object construction omits fields whose value is `undefined`.
- Object construction MUST preserve `null` fields.

Object equality is based on key set and values, not insertion order.

> **Implementation note.** Implementations whose host string comparison differs from Unicode code-point order, such as UTF-16 code-unit ordering for supplementary-plane characters, MUST still expose keys in the BEX-defined order under the Fixture Profile.

### 3.6 Lists (normative)

BEX lists are ordered sequences.

- Indexing is zero-based.
- An index MUST be a non-negative integer.
- Out-of-range indexes read as `undefined`.
- Lists MUST NOT contain `undefined`.
- Lists preserve order and multiplicity.

### 3.7 Truthiness (normative)

Truthiness is used by `$and`, `$or`, `$not`, `$truthy`, `$empty`, `$if`, `$choose`, and `$coalesce`.

| Value | Truthy? |
|---|---|
| `undefined` | false |
| `null` | false |
| boolean | its boolean value |
| text | true if non-empty |
| integer or number | true, including zero |
| object | true if it has at least one key |
| list | true if it has at least one item |

`$empty(x)` is equivalent to `not truthy(x)`.

### 3.8 Equality (normative)

`$eq` and `$ne` use BEX value equality.

- `undefined` equals only `undefined`.
- `null` equals only `null`.
- Booleans compare by boolean value.
- Text compares by exact text value.
- Numbers compare numerically when both operands are numeric scalar values; integer `1` and decimal `1.0` compare equal as numbers.
- Lists compare by length and element equality in order.
- Objects compare by their key set and value equality for each key.
- Node-backed values compare through their visible BEX value behavior. Implementations MUST make equality deterministic.

BEX value equality is an execution-level relation. It is not Blue node identity, not BlueId equality, and not canonical Blue scalar identity. When Blue identity matters, hosts must compare Blue nodes or BlueIds under the Blue Language rules.

### 3.9 Conversions (normative)

The type conversion operators use these conversions:

| Operator | Conversion |
|---|---|
| `$text` | `undefined` and `null` become `""`; scalars use the scalar text representation (§3.9.1); non-scalars fail. |
| `$integer` | exact integer conversion; non-integral decimal and invalid text fail. |
| `$number` | decimal numeric conversion from integer, number, or numeric text; invalid text fails. |
| `$boolean` | `undefined` and `null` become false; booleans keep value; text `"true"` and `"false"` map to booleans; other values use truthiness. |
| `$object` | `undefined` and `null` become `{}`; objects stay objects; other values fail. |
| `$list` | `undefined` and `null` become `[]`; lists stay lists; other values fail. |

### 3.9.1 Scalar text representation (normative)

`$text`, `$concat`, `$join`, `$pointerJoin`, `$startsWith`, `$sliceAfter`, `$fail` message conversion, and gas size estimation use the same BEX scalar text representation.

`undefined` and `null` convert to empty text only where text conversion is explicitly allowed.

Scalar text rendering is:

- booleans render exactly as `true` and `false`;
- integers render as canonical decimal text with optional leading `-`, no leading `+`, and no leading zeros except `0`;
- decimals render deterministically, locale-independently, and fixture-stably;
- under the BEX 1.0 Fixture Profile, decimal rendering follows Java `BigDecimal.toString()` / `String.valueOf`-compatible rendering;
- text values render as themselves with no Unicode normalization.

No locale-sensitive formatting, grouping separators, currency formatting, timezone formatting, or host-specific number formatting is allowed.

---

## 4. Compilation

### 4.1 Compile-time failures (normative)

Compilation MUST fail for:

- unknown expression operators;
- unknown statement operators;
- malformed statement objects;
- null or empty statement items;
- unknown constants referenced by `$const`;
- unknown variables referenced by `$var`, including object-form `$var.name`;
- unknown functions referenced by `$call` or `entry`;
- missing function call arguments;
- extra function call arguments;
- reserved names in user-defined name containers;
- BEX expressions inside static Blue definition fields;
- dynamic or computed patterns for function arguments and `$is.pattern`;
- recursive function calls, whether direct or indirect;
- invalid operator body shapes that are statically known;
- unsupported `$intrinsic.type` BlueIds for the active processor registry;
- invalid `$isKind.kind` values;
- duplicate binding names within a single collection query or `$reduce` binding set;
- `$let.order` that omits, duplicates, or references names outside `$let.vars`.

Compile-time failures MUST expose an error class. They MUST include the operator name when the failing operator is known. They MUST include a source path within the BEX source document when the source path is known. They MUST NOT fabricate a path or operator name if unavailable.

### 4.2 Root compilation (normative)

A compiled program contains a root function. The root function either evaluates a root expression, executes root statements, or invokes an entry function.

Every execution invokes the root function and therefore charges function-call gas (§12).

### 4.3 Function compilation (normative)

A user function definition MAY contain:

| Field | Shape | Meaning |
|---|---|---|
| `args` | plain object | Required argument patterns by name. |
| `expr` | expression | Expression function body. |
| `do` | list of statements | Statement function body. |

If `expr` is present, the function is an expression function. Otherwise, the function is a statement function using `do`. If `do` is absent or empty, the statement function has an empty statement list and returns the default result value (§7.9).

All declared arguments are required. Extra call arguments are forbidden. Function argument names are matched by name, not by position.

Function argument patterns are static Blue nodes. The compiler MUST reject BEX expressions inside argument patterns.

### 4.4 Recursion (normative)

BEX functions MUST NOT be recursive. The compiler MUST reject direct or indirect cycles in the static call graph.

Calls hidden inside `$literal` are not executable calls and MUST NOT contribute to the call graph. Calls inside static `$is.pattern` content are invalid because static patterns cannot contain BEX expressions.

### 4.5 Local variables (normative)

Local variables are function-frame slots.

- `$let` declares or assigns a local variable in the current function frame.
- `$set` assigns an existing local variable and MUST fail compilation if the variable has not been declared in the current function's compile scope.
- `$let.vars` may declare or assign multiple local variables with either parallel or explicit sequential evaluation (§7.3).
- Function calls execute in a separate frame.
- Caller local variables MUST NOT leak into callee frames.
- Callee variables MUST NOT leak back into caller frames, except through the returned value, changeset, or events.

Nested statement blocks do not create separate lexical scopes unless an implementation declares an extension profile. Duplicate loop binding names within a single `$forEach` statement MUST be rejected.

Collection query binding slots (`item`, optional `key`, optional `index`, and `$reduce.acc`) are temporary for the expression evaluation. If they reuse an outer local variable name, the previous slot value MUST be restored after the collection expression completes or fails.

### 4.6 Static versus dynamic operands (normative)

Many operators accept either static scalar shorthand or dynamic expressions. Implementations MUST distinguish:

- a static omitted path, which may intentionally mean a default root path;
- a dynamic expression that evaluates to `undefined` or `null`, which MUST fail when a path, key, operation, binding name, or step name is required.

Dynamic text operands that evaluate to `undefined` or `null` MUST fail for:

```text
$get.key, $objectSet.key, $hasKey.key, $binding.name, $steps.step,
$appendChange.op, $pointerSet.op
```

Dynamic pointer operands that evaluate to `undefined` or `null` MUST fail.

---

## 5. Execution Context and Document Views

### 5.1 Execution context (normative)

A BEX execution context MAY provide:

| Context value | Meaning |
|---|---|
| `documentScope` | Current document pointer scope for document-relative pointers. |
| `rootDocument` | Canonical input document. |
| resolved document view | Resolved input document view, if supported. |
| `event` | Current event value. |
| `currentContract` | Current contract value. |
| `steps` | Map of named prior step values. |
| `bindings` | Map of named binding values. |
| `gasLimit` | Maximum gas before exhaustion, or unlimited if negative. |
| `gasSchedule` | Deterministic gas schedule. |

Missing context values read as `undefined` unless an operator specifies a different error.

### 5.2 Document view (normative)

`$document` reads from the current document view.

By default, `$document` reads the canonical view:

```yaml
$document: /status
```

A resolved view is requested with:

```yaml
$document:
  path: /status
  view: resolved
```

Only the exact view value `resolved` selects the resolved view. Other or absent view values select the canonical view.

If the host cannot provide a requested resolved view, execution MUST fail or return a deterministic host-defined error. It MUST NOT silently substitute unrelated content.

### 5.3 Document scope (normative)

Document pointers are resolved relative to `documentScope` unless they are absolute JSON Pointers beginning with `/`.

- A missing or empty static document path resolves to the current document scope.
- An absolute path resolves from the root document.
- A relative path appends its pointer segments to the current document scope.

### 5.4 Host values and immutability (normative)

Host-provided Blue nodes used by BEX MUST be immutable for the duration of execution, or MUST be snapshotted before execution. A host MUST NOT allow mutable shared nodes to change while a BEX program is executing.

An implementation MAY expose optimized trusted immutable cursors, but only when the host guarantees immutability.

---

## 6. Expressions

### 6.1 Expression evaluation (normative)

Each expression evaluation MUST:

1. charge `expressionBase` gas;
2. increment the expression-evaluation metric;
3. evaluate according to the operator or literal semantics;
4. return a BEX value or fail with a runtime error.

Source wrappers and compile-time structures do not themselves consume gas unless they execute as expressions.

### 6.1.1 Operand evaluation order (normative)

Unless an operator explicitly short-circuits or defines lazy behavior, evaluated operands MUST be evaluated in a deterministic order.

- List operand sequences are evaluated left to right by list index.
- List literal items are evaluated left to right by list index.
- Object literal fields are evaluated in BEX object key exposure order, not host map iteration order.
- `$call.args` expressions are evaluated in lexicographic argument-name order, not source map order.

The following operands remain lazy or selected-only according to their operator semantics:

```text
$and, $or, $coalesce, $choose, $if,
$listGet.default, $pointerGet.default,
$appendChange.val for remove, $pointerSet.val for remove,
$returnIf.expr when cond is false, $failIf.message when cond is false,
$some/$find/$findEntry/$includes items after a short-circuit match
```

Skipped lazy operands produce no side effects and consume no gas.

### 6.2 Literal expressions (normative)

Scalars, lists, and non-operator objects are expressions.

The explicit literal helper operators are:

| Operator | Semantics |
|---|---|
| `$null` | Return BEX `null`. |
| `$emptyObject` | Return an empty object. |
| `$emptyList` | Return an empty list. |

- A scalar literal evaluates to the corresponding scalar BEX value, or `null` for a null literal.
- A list literal evaluates each item and returns a list. If any item evaluates to `undefined`, list construction MUST fail.
- An object literal evaluates ordinary fields in BEX object key exposure order and returns an object. Fields whose values evaluate to `undefined` are omitted.
- Blue language metadata fields may be preserved as Blue output fields when the object is later converted to a Blue node.

### 6.3 Read expressions (normative)

| Operator | Body | Semantics |
|---|---|---|
| `$document` | pointer or `{path, view}` | Read canonical or resolved document view at a document pointer. |
| `$binding` | `name/path` or `{name, path}` | Read named host binding at value-local path. |
| `$event` | pointer | Read current event at value-local path. |
| `$steps` | `step.path` or `{step, path}` | Read named prior step value at value-local path. |
| `$currentContract` | pointer | Read current contract value at value-local path. |
| `$var` | name or `{name, path?}` | Read local variable slot, optionally at a value-local path. |
| `$const` | name or `{name, path?}` | Read compile-time constant, optionally at a value-local path. |
| `$get` | `{object, key}` | Read object key; missing or non-object reads as `undefined`. |
| `$changeset` | ignored | Return accumulated changeset as a BEX value. |
| `$events` | ignored | Return accumulated events as a BEX value. |
| `$resultValue` | pointer | Read document value after accumulated changes have been overlaid. |

`$binding` short form splits at the first `/`. The text before the first slash is the binding name. The slash and following text are the path. If no slash appears, the path is `/`.

`$steps` short form splits at the first `.`. The text before the first dot is the step name. The dot and following text are converted to a path beginning with `/`. If no dot appears, the path is `/`.

`$var` and `$const` object forms have a static `name` and optional value-local `path`:

```yaml
$var:
  name: request
  path: /summary
```

```yaml
$const:
  name: Policy/minimumAmount
  path: /amount
```

The `name` operand is static and MUST NOT be computed. Dynamic variable names and dynamic constant names are not part of BEX 1.0. The `path` operand MAY be static text or a dynamic expression that evaluates to pointer text. A missing path target returns `undefined`.

There is no BEX 1.0 shorthand that splits `$var: request/summary` or `$const: Policy/minimumAmount`. Variable and constant names may contain `/`; object-form `path` exists to avoid changing the meaning of such programs.

### 6.4 Type and conversion expressions (normative)

| Operator | Semantics |
|---|---|
| `$unwrap` | Repeatedly reads `value` while the current value is an object with a defined `value` field. |
| `$is` | Blue type/shape match of `node` against static `pattern`. |
| `$kind` | Return the visible BEX runtime kind of a value. |
| `$isKind` | Return whether a value's visible BEX runtime kind is in a static kind set. |
| `$text` | Convert to text (§3.9). |
| `$integer` | Convert to exact integer (§3.9). |
| `$number` | Convert to decimal number (§3.9). |
| `$boolean` | Convert to boolean (§3.9). |
| `$object` | Convert `undefined`/`null` to `{}` or pass through object; otherwise fail. |
| `$list` | Convert `undefined`/`null` to `[]` or pass through list; otherwise fail. |

`$is` body MUST contain `node` and static `pattern` operands. The pattern MUST NOT contain BEX expressions.

`$is` evaluates `node` first. If evaluating `$is.node` itself fails, that runtime failure propagates and is not converted to `false`.

If `$is.node` evaluates successfully but the resulting value is `undefined`, `$is` returns `false`. If conversion of the successfully evaluated value to a Blue node fails, `$is` returns `false`. If the Blue type matcher returns no match, `$is` returns `false`.

If the static pattern is malformed or contains BEX expressions, compilation fails. Host, provider, or type-resolution failures required to evaluate the pattern deterministically are errors according to the host's declared Blue type matcher profile; implementations MUST NOT silently treat unavailable type definitions as a successful non-match unless the type matcher profile explicitly does so. Function argument matching uses the same type matcher boundary (§8.3).

`$kind` returns one of:

```text
undefined, null, text, integer, double, boolean, object, list
```

`$kind` exposes the visible BEX value kind, not an implementation artifact such as a node cursor class. Node-backed and frozen values MUST report the kind they behave as in BEX.

`$isKind` has body `{val, kind}`. The `kind` operand is static authored data and MUST be either a single kind text value or a list of kind text values. Unknown kind names MUST fail compilation. The `val` operand is evaluated at runtime.

Example:

```yaml
$isKind:
  val:
    $event: /message/request/amount
  kind: [integer, double]
```

### 6.5 String expressions (normative)

| Operator | Body | Semantics |
|---|---|---|
| `$concat` | list of operands | Convert each operand to text and concatenate. |
| `$pointerJoin` | list of segment operands | Convert each segment to text, JSON-Pointer-escape it, and join as an absolute pointer. No segments returns `/`. |
| `$join` | `{list, separator}` | Join list items converted to text using separator text. |
| `$split` | `{text, separator, limit?}` | Split text by non-empty separator. `limit` is optional; `-1` means no limit. |
| `$startsWith` | two operands | True if the first text starts with the second text. |
| `$sliceAfter` | two operands | If first text starts with second text, return the suffix after that prefix; otherwise return empty text. |

`$split.separator` MUST NOT be empty. A missing `$split.limit` is equivalent to `-1`.

`$split.limit` semantics are:

- `-1` means no limit and preserves trailing empty parts;
- `n > 0` returns at most `n` parts;
- for `n > 0`, the first `n - 1` separator occurrences split normally, and the final part contains the remaining suffix, including separators that were not consumed;
- `1` returns the whole input as a single-item list;
- `0` or any value less than `-1` MUST fail;
- if the separator does not occur, the result is a one-item list containing the input text.

Examples:

```yaml
{ $split: { text: "a,b,c", separator: ",", limit: 2 } }
# => ["a", "b,c"]

{ $split: { text: "a,", separator: "," } }
# => ["a", ""]

{ $split: { text: "a,", separator: ",", limit: 1 } }
# => ["a,"]
```

`$pointerJoin` MUST escape `~` as `~0` and `/` as `~1`.

### 6.6 Logic and comparison expressions (normative)

| Operator | Semantics |
|---|---|
| `$eq` | BEX equality of exactly two operands. |
| `$ne` | Negation of `$eq`. |
| `$gt`, `$gte`, `$lt`, `$lte` | Numeric comparison of exactly two operands using decimal numeric conversion. |
| `$and` | Left-to-right truthy conjunction with short-circuit. Empty operand list returns `true`. |
| `$or` | Left-to-right truthy disjunction with short-circuit. Empty operand list returns `false`. |
| `$not` | Truthy negation. |
| `$truthy` | Return truthiness as boolean. |
| `$empty` | Return inverse truthiness as boolean. |
| `$isEmpty` | Alias for `$empty`. |
| `$exists` | Return false only for `undefined`; true otherwise. |
| `$coalesce` | Return the first truthy operand; if none is truthy, return `undefined`. |
| `$default` | Alias for `$coalesce`. |

`$and`, `$or`, and `$coalesce` MUST NOT evaluate operands after their result is determined. Unevaluated operands consume no gas and produce no errors.

`$isEmpty` is an exact alias for `$empty`. It exists to avoid ambiguity with Blue empty-placeholder authoring syntax when an empty check appears in a list operand.

### 6.7 Numeric expressions (normative)

| Operator | Semantics |
|---|---|
| `$add` | Exact integer addition. |
| `$subtract` | Exact integer subtraction. |
| `$multiply` | Exact integer multiplication. |
| `$divide` | Exact integer division. |

Numeric arithmetic operators use exact integer conversion for all operands. Non-integer numeric values and invalid integer text MUST fail. Division by zero MUST fail. Division with a non-zero remainder MUST fail.

A one-operand numeric expression returns that operand converted to integer.

### 6.8 Object and list expressions (normative)

| Operator | Body | Semantics |
|---|---|---|
| `$keys` | expression | Return sorted object keys as a list; non-objects return `[]`. |
| `$entries` | expression | Return sorted object entries as `{key, val}` objects. |
| `$size` | expression | List length, object field count, scalar `1`, or `0` for `undefined`/`null`. |
| `$listGet` | `{list, index, default?}` | Read list index; use default only when missing. |
| `$listConcat` | list of list operands | Concatenate lists. All operands MUST be lists. |
| `$merge` | list of object operands | Shallow merge objects left to right; later keys win. |
| `$objectSet` | `{object, key, val}` | Set or remove object key. Undefined `val` removes the key. |
| `$pointerGet` | `{object, path, default?}` | Read a value-local JSON Pointer; evaluate default only when missing. |
| `$pointerSet` | `{object, path, op?, val?}` | Set or remove a value-local JSON Pointer. |
| `$map` | `{in, item, key?, index?, expr}` | Project each list item or sorted object value to a list. |
| `$filter` | `{in, item, key?, index?, where}` | Keep matching list items or object fields. |
| `$flatMap` | `{in, item, key?, index?, expr}` | Project each item to a list and concatenate. |
| `$reduce` | `{in, acc, init, item, key?, index?, expr}` | Fold a list or sorted object values into an accumulator. |
| `$some` | `{in, item, key?, index?, where}` | Return true on the first truthy match; otherwise false. |
| `$find` | `{in, item, key?, index?, where}` | Return the first matching item/value, or `undefined`. |
| `$findEntry` | `{in, item, key?, index?, where}` | Return the first matching entry object, or `undefined`. |
| `$includes` | `{list, val}` | Return whether a list contains `val` by BEX equality. |
| `$hasKey` | `{object, key}` | Return whether an object has a non-undefined value for `key`. |
| `$objectFromEntries` | list expression | Build an object from `{key, val}` entries. |

`$objectSet` treats an `undefined` or `null` object operand as `{}`. It MUST fail for scalar base values.

`$pointerSet.op` defaults to `set`. The only valid operations are `set` and `remove`. `remove` MUST NOT evaluate its `val` operand. `set` MUST evaluate `val`.

When `$pointerSet` needs to create missing intermediate containers, it creates objects. It MUST fail if an existing intermediate value is scalar or otherwise incompatible with traversal.

`$size` is a collection/cardinality helper. For scalar text it returns `1`, not text length. A dedicated text code-point length operator is a candidate for a later BEX revision.

#### 6.8.1 Collection query expressions (normative)

Collection query expressions accept either a list or an object in `in`. Non-list and non-object inputs MUST fail unless a specific operator says otherwise.

For list input:

- iteration order is list index order;
- `item` receives the list item;
- `index`, when present, receives the zero-based integer index;
- `key`, when present, receives `undefined`.

For object input:

- iteration order is lexicographic Unicode code-point order of object keys;
- `item` receives the field value;
- `key`, when present, receives the object key text;
- `index`, when present, receives the zero-based ordinal in sorted-key order.

The binding names `item`, optional `key`, optional `index`, and `$reduce.acc` within one collection expression MUST be distinct when present. Collection expression bindings MUST restore any previous slot values after the expression completes or fails.

`$map` evaluates `expr` for each item and returns a list of results. Object input still returns a list, in sorted-key order.

`$filter` evaluates `where` for each item. For list input it returns a filtered list preserving item order. For object input it returns a filtered object preserving BEX object key exposure semantics.

`$flatMap` evaluates `expr` for each item. Each result MUST be a list. The returned value is a concatenated list of all projected lists.

`$reduce` evaluates `init` once after evaluating `in`, assigns it to `acc`, then evaluates `expr` once per item. After each iteration, the expression result becomes the next accumulator value. The final accumulator is returned. For empty input, the `init` value is returned.

`$some`, `$find`, `$findEntry`, and `$includes` are short-circuiting. Items after a determined result MUST NOT be evaluated and consume no gas.

`$some` returns true for the first truthy `where`, otherwise false.

`$find` returns the first item/value whose `where` is truthy, otherwise `undefined`.

For list input, `$findEntry` returns:

```yaml
val: <item>
index: <zero-based-index>
```

For object input, `$findEntry` returns:

```yaml
key: <object-key>
val: <field-value>
index: <zero-based-sorted-key-index>
```

`$includes.list` MUST evaluate to a list. It evaluates `val` once, then compares each list item using BEX equality. It returns true on the first equal item, otherwise false.

`$hasKey.object` returns false for non-object inputs. For object inputs it returns true when reading `key` returns a non-`undefined` value. The `key` operand is a static or dynamic text operand and follows dynamic text failure rules (§4.6).

`$objectFromEntries` requires a list of object entries. Each entry MUST be an object. Each entry key is read from `key`; `undefined` or `null` keys MUST fail. Keys are converted to text. Each entry value is read from `val`; an `undefined` value removes/omits that key from the output object. Duplicate keys use the last non-undefined value unless a later undefined value removes the key.

### 6.9 Result helper expressions (normative)

`$changeset` returns the accumulated changeset as a list of patch objects.

`$events` returns the accumulated event list.

`$resultValue` reads the input document after applying all accumulated patches in order through the result overlay model (§10). It charges `resultValueRead` gas in addition to expression gas.

### 6.10 Control expressions (normative)

| Operator | Body | Semantics |
|---|---|---|
| `$choose` | `{cond, then, else?}` | Evaluate `cond`; if truthy evaluate `then`, otherwise evaluate `else` or return `undefined`. |
| `$call` | `{function, args}` | Invoke a user-defined function with named arguments. |
| `$intrinsic` | object with static `type` and payload fields | Invoke a registered host intrinsic processor by the BlueId of `type`. |
| `$literal` | any | Return literal body without nested BEX interpretation, subject to §2.6. |
| `$null`, `$emptyObject`, `$emptyList` | ignored | Return explicit null, empty object, or empty list. |

`$call` evaluates argument expressions, validates each argument against its declared static pattern, and then invokes the callee in a new frame. Argument validation failure is a runtime error.

`$call` may also appear as a statement; statement-form `$call` invokes the function for effects and discards its return value (§7.2).

### 6.11 `$intrinsic` (normative)

`$intrinsic` is BEX's explicit host capability boundary. It invokes a host-registered processor keyed by the BlueId of the static `type` field.

Example:

```yaml
$intrinsic:
  type:
    blueId: CommonCryptoEd25519Verify
  publicKey:
    $const: trustedSignerPublicKey
  message:
    $event: /message/canonicalBytes
  signature:
    $event: /message/signature
```

Rules:

- `$intrinsic` body MUST be an object.
- `type` is REQUIRED and MUST be static authored Blue data. BEX expressions inside `type` MUST fail compilation.
- The implementation resolves or computes the BlueId of `type` using the declared Blue Language profile.
- The resolved BlueId MUST have a registered intrinsic processor in the active BEX runtime/profile. Otherwise compilation MUST fail.
- Payload fields are the ordinary object properties beside `type`. The `type` field itself is not passed as a payload field; processors receive it separately as static type data.
- Payload field expressions are evaluated normally in deterministic object key exposure order. Fields evaluating to `undefined` are omitted.
- A processor returns one BEX value. A host API that permits a null implementation return MUST normalize it to BEX `undefined` or fail deterministically.
- A processor is responsible for deterministic gas accounting for its own work (§12.7).

`$intrinsic` MUST NOT be used for arbitrary non-deterministic host calls in a portable BEX profile. Standard intrinsics such as signature verification MUST define their exact input bytes, failure behavior, and conformance vectors outside the BEX program text.

---

## 7. Statements

### 7.1 Statement execution (normative)

Each statement execution MUST:

1. charge `statementBase` gas;
2. increment the statement-execution metric;
3. execute according to the statement operator;
4. either proceed to the next statement, return, or fail.

A statement list executes in order until it completes, returns, or fails.

### 7.2 Statement operators (normative)

| Operator | Body | Semantics |
|---|---|---|
| `$let` | `{name, expr}` or `{vars, order?}` | Declare or assign one or more local variables. |
| `$set` | `{name, expr}` | Assign existing local variable. |
| `$if` | `{cond, then?, else?}` | Execute selected statement list. |
| `$forEach` | `{in, item, key?, index?, do}` | Iterate list or object. |
| `$appendChange` | `{op, path, val?}` | Append one document patch. |
| `$appendChanges` | expression | Append a list of patch entries. |
| `$appendEvent` | expression | Append one event value. |
| `$appendEvents` | expression | Append a list of event values. |
| `$call` | `{function, args}` | Invoke function for effects and discard return value. |
| `$return` | expression or empty | Return from current function. |
| `$returnIf` | `{cond, expr?}` | Return from current function when `cond` is truthy. |
| `$fail` | expression or `{message}` | Throw a runtime failure with message text. |
| `$failIf` | `{cond, message}` | Throw a runtime failure when `cond` is truthy. |

### 7.3 `$let` and `$set` (normative)

Single-bind `$let` evaluates `expr` and assigns it to `name` in the current frame. Reusing an existing name assigns the existing slot.

Multi-bind `$let` uses `vars`:

```yaml
$let:
  vars:
    a: 1
    b: 2
```

Without `order`, `$let.vars` is parallel: all binding expressions are evaluated against the frame as it existed before the `$let`, then all variables are assigned. Implementations MAY sort variable names to make evaluation deterministic, but sorted evaluation MUST NOT create intra-batch dependencies.

With `order`, `$let.vars` is sequential:

```yaml
$let:
  order: [request, summary]
  vars:
    request:
      $event: /message/request
    summary:
      $var:
        name: request
        path: /summary
```

`order` MUST list every key in `vars` exactly once. Later ordered bindings may read earlier ordered bindings.

`$set` evaluates `expr` and assigns an existing slot. Compilation MUST fail if the name is not known in the current function compile scope.

### 7.4 `$if` (normative)

`$if` evaluates `cond`. If truthy, it executes `then`; otherwise it executes `else`. Only the selected branch executes and consumes gas.

A missing `then` or `else` branch is an empty statement list.

### 7.5 `$forEach` (normative)

`$forEach.in` MUST evaluate to a list or object.

For a list:

- `item` receives each item;
- `index`, when provided, receives the zero-based integer index;
- `key`, when provided, receives `undefined`.

For an object:

- keys are visited in lexicographic order over Unicode code points of the unescaped key text;
- if `key` is provided, `key` receives the key text and `item` receives the field value;
- if `key` is not provided, `item` receives an object of the form `{ key: <key>, val: <value> }`;
- `index`, when provided, receives `undefined`.

The loop body executes for each element. Each iteration charges `forEachItem` gas in addition to statement and body gas.

The names `item`, `key`, and `index` within one `$forEach` statement MUST be distinct when present.

BEX 1.0 iteration has no loop-local `break` or loop-local `continue`. `$return` exits the current function, not merely the loop. `$fail` aborts execution.

### 7.6 `$appendChange` (normative)

`$appendChange` appends one patch entry to the changeset accumulator.

Valid operations are:

```text
add, replace, remove
```

Rules:

- `op` is a required text operand.
- `path` is a document pointer operand and is resolved relative to the document scope when not absolute.
- `add` and `replace` require a non-`undefined` `val`.
- `remove` MUST NOT evaluate `val`.
- The appended patch preserves author order. Duplicate paths are allowed and are not coalesced.

### 7.7 `$appendChanges` (normative)

`$appendChanges` evaluates its body as a list of patch entries. Each entry is validated as if supplied to `$appendChange`.

Invalid entries, bad operations, missing values for `add`/`replace`, or non-list inputs MUST fail. Successfully appended entries preserve list order.

### 7.8 `$appendEvent` and `$appendEvents` (normative)

`$appendEvent` evaluates its body and appends the resulting value to the event accumulator. The event value MUST NOT be `undefined`. Events need not be objects.

`$appendEvents` evaluates its body as a list and appends each item in order. Each item MUST NOT be `undefined`.

### 7.9 `$return` (normative)

`$return` returns from the current function.

If `$return` has an expression body, that expression is the returned value. If `$return` is empty or null, the default result value is returned.

At the root statement list, `$return` returns from the root function and therefore exits the whole BEX program. Statements after that root `$return` MUST NOT execute.

The default result value is an object with:

```yaml
changeset: <current changeset as value>
events: <current events as value>
```

If a statement function completes without an explicit `$return`, it returns the same default result value.

### 7.10 `$returnIf` and `$failIf` (normative)

`$returnIf` evaluates `cond`. If truthy, it evaluates optional `expr` and returns from the current function with that value. If `expr` is absent, it returns the default result value. If `cond` is falsy, `expr` MUST NOT be evaluated.

The return payload field is named `expr`. `value` is not a valid `$returnIf` payload field because `value` is a Blue scalar-wrapper field in authored Blue data.

`$failIf` evaluates `cond`. If truthy, it evaluates `message`, converts it to text, and raises a runtime error. If `cond` is falsy, `message` MUST NOT be evaluated.

### 7.11 `$fail` (normative)

`$fail` raises a runtime error. If the body is an object with a `message` field, the `message` operand supplies the error message. Otherwise, the body expression supplies the message. The message is converted to text.

---

## 8. Functions, Constants, and Static Patterns

### 8.1 Constants (normative)

`constants` is a plain name container. A constant value is compiled as static literal content. `$const` references a named constant.

A `$const` reference to an unknown constant MUST fail compilation.

Constants are immutable during execution.

### 8.2 Functions (normative)

`functions` is a plain name container. A function name maps to a function definition.

Function names are static. `$call.function` MUST resolve to a known function at compile time. Dynamic function dispatch is not part of BEX 1.0.

### 8.3 Function arguments (normative)

Function `args` is a plain name container whose values are static Blue type or shape patterns.

A call MUST supply exactly the declared argument names:

- missing declared argument: compile error;
- extra argument: compile error;
- unknown function: compile error;
- invalid reserved argument name: compile error.

At runtime, after each argument expression is evaluated, the value MUST match the declared Blue pattern if the pattern is non-empty. Mismatch is a runtime error.

Function argument pattern matching uses the same Blue type matcher boundary as `$is` (§6.4). If argument expression evaluation itself fails, that failure propagates. If an argument evaluates successfully but conversion or type matching returns non-match, argument validation fails as a runtime error. Host, provider, and type-resolution failures follow the declared Blue type matcher profile.

### 8.4 Static `$is.pattern` (normative)

`$is.pattern` is a static Blue node. It MUST NOT contain BEX expressions.

`$is.node` is an expression and MAY contain BEX.

### 8.5 Empty patterns (normative)

An empty or null Blue pattern matches any non-`undefined` value. `undefined` does not match a pattern.

### 8.6 Pattern labels (normative)

Blue `name` and `description` semantics are inherited from the Blue type matcher. They are matcher-neutral when Blue Language type matching treats them as matcher-neutral.

---

## 9. Pointers

### 9.1 JSON Pointer syntax (normative)

BEX pointer strings use JSON Pointer syntax with `/`-separated segments. Implementations MUST support `~0` for `~` and `~1` for `/` in pointer segments.

A pointer that does not begin with `/` may be interpreted relative to a scope depending on pointer kind.

### 9.2 Pointer kinds (normative)

BEX distinguishes document pointers from value-local pointers.

| Pointer kind | Used by | Scope |
|---|---|---|
| Document pointer | `$document`, `$resultValue`, `$appendChange.path`, `$appendChanges` entry path | Relative to current document scope unless absolute. |
| Value-local pointer | `$event`, `$currentContract`, `$steps.path`, `$binding.path`, `$pointerGet.path`, `$pointerSet.path` | Relative to the root of the operand value. |

### 9.3 Static pointer defaults (normative)

A missing or empty static pointer MAY intentionally mean the root/default path.

- Static document path omitted or empty: current document scope.
- Static value path omitted or empty: `/`.

### 9.4 Dynamic pointer failures (normative)

A dynamic pointer expression that evaluates to `undefined` or `null` MUST fail. It MUST NOT be silently interpreted as `/`.

Dynamic pointer text is canonicalized as JSON Pointer text. Relative dynamic document pointers are resolved against document scope. Relative dynamic value pointers are made value-local by prefixing `/`.

### 9.5 `$pointerJoin` (normative)

`$pointerJoin` builds an absolute JSON Pointer from unescaped segment values.

```yaml
$pointerJoin: [rooms, "12/34", "a~b"]
# => /rooms/12~134/a~0b
```

No segments returns `/`.

---

## 10. Changesets, Events, and Result Overlay

### 10.1 Changeset entries (normative)

A changeset is an ordered list of patch entries.

A patch entry has:

| Field | Meaning |
|---|---|
| `op` | `add`, `replace`, or `remove`. |
| `path` | Absolute document pointer after scope resolution. |
| `val` | Patch value for `add` and `replace`; absent for `remove`. |

Patch entries are accumulated. BEX does not itself persist or apply them to the host document.

### 10.2 Patch order and duplicates (normative)

Patch entries MUST remain in append order. Duplicate paths MUST be preserved. Implementations MUST NOT coalesce, reorder, or discard patches in the accumulator.

### 10.3 Events (normative)

Events are ordered BEX values. Events need not be Blue objects unless the host imposes such a requirement. BEX MUST preserve event append order.

### 10.4 `$resultValue` overlay (normative)

`$resultValue` reads from a transient overlay formed by applying accumulated patches, in order, to the canonical document view.

The overlay is a BEX execution view. It does not mutate the host document.

Rules:

- A later patch to the same path is visible to later `$resultValue` reads.
- A read of a parent object reflects descendant patches.
- A parent replacement followed by a child replacement is applied in order.
- Removing a child makes that child read as `undefined`.
- Adding to a missing parent creates intermediate object containers for overlay purposes.
- List index replacement is supported.
- List index removal is non-shifting in the overlay model; later indexes retain their positions.
- Root replacement replaces the overlay root.
- Root removal makes the overlay root `undefined`.

### 10.4.1 Sparse list overlay views (normative)

A list-index remove patch in `$resultValue` creates a sparse overlay slot at that index.

- Direct reads of the removed index return `undefined`.
- Later indexes retain their original positions.
- `$size` of such an overlay list returns the overlay list's positional extent, including removed slots.
- A sparse overlay list is a BEX overlay view, not a dense BEX list literal.
- Converting a sparse overlay list to a Blue list MUST fail if any indexed slot in `0..size-1` is `undefined`, because Blue/BEX output lists cannot contain `undefined`.
- Programs that need portable Blue output after a non-shifting removal MUST return specific paths, construct a dense list explicitly, or leave the change in the changeset for the host to apply.

### 10.5 Result value output (normative)

The final `value` returned by a program is independent of the changeset and event accumulators unless the program explicitly returns `$changeset`, `$events`, `$resultValue`, or the default result value.

---

## 11. Blue Output Boundary

### 11.1 Purpose (normative)

BEX values are not automatically Blue nodes. When a host converts a BEX value to Blue content, it MUST apply the Blue output boundary rules in this section.

### 11.2 Conversion to Blue node (normative)

Conversion rules:

| BEX value | Blue output |
|---|---|
| `undefined` | invalid as root output and invalid as list item. |
| `null` | empty Blue node. |
| scalar | Blue scalar node with `value`. |
| list | Blue list node with `items`, converting each item. |
| object without Blue language fields | Blue object with ordinary fields. |
| object with Blue language fields | Blue node using reserved field semantics. |
| node cursor / frozen node | the corresponding Blue node, cloned or referenced immutably according to host policy. |
| overlay | Materialized through its visible BEX value behavior; conversion fails if materialization produces root `undefined`, an undefined list item, an invalid Blue node shape, or unsupported sparse list slots. |

Numeric output conversion MUST preserve or choose the Blue numeric scalar type deterministically:

- exact BEX integers convert to Blue `Integer`;
- non-integer BEX decimals convert to Blue `Double` only through deterministic finite binary64 conversion, or conversion fails if the value cannot be represented under the declared host profile;
- explicit Blue-shaped output with `type` and `value` follows the Blue Language scalar rules.

### 11.3 Pure reference rule (normative)

If a converted object contains `blueId`, it MUST contain exactly `blueId` and no sibling fields. A mixed `blueId` object MUST fail output conversion.

### 11.4 Payload-kind rule (normative)

A converted Blue node MUST NOT combine incompatible payload kinds. In particular:

- `value` MUST NOT be combined with `items` or ordinary child fields;
- `items` MUST NOT be combined with ordinary child fields;
- `properties` MUST NOT be used as a Blue object wrapper.

### 11.5 Schema compatibility (normative)

Portable BEX output MUST use Blue Language `schema` and only schema keywords defined by the declared target Blue Language profile.

Under the strict BEX 1.0 Host Boundary Profile, computed output containing `constraints`, a schema containing `allowMultiple` or `options`, or any schema key outside the target Blue Language profile MUST fail output conversion.

A declared Java Compatibility Output Extension Profile MAY accept `constraints` as a compatibility alias for `schema`, and MAY accept compatibility schema keys only when the underlying Blue implementation supports them.

Output accepted only by the Java Compatibility Output Extension Profile is not portable Blue Language 1.0 output.

If both `schema` and `constraints` are present, conversion MUST fail in all profiles.

### 11.6 `blue` output boundary (normative)

Computed output containing `blue` MUST fail under the strict BEX 1.0 Host Boundary Profile unless the host explicitly declares an authoring-output extension profile.

Such an extension profile means the output is a Blue Source Document candidate, not direct portable BlueId Input or stored resolved content.

BEX MUST NOT silently treat computed `blue` as ordinary data.

### 11.7 List-control output fields (normative)

Computed output MUST NOT emit `$previous`, `$pos`, or unsupported list-control fields as Blue node-control metadata unless a declared extension profile defines their conversion. The strict BEX 1.0 Host Boundary Profile rejects such output by default.

---

## 12. Gas Accounting

### 12.1 Goal (normative)

BEX gas accounting is deterministic. For the same compiled program, context, input values, gas schedule, and immutable host views, a conforming implementation MUST consume the same gas and either produce the same result or fail at the same gas exhaustion boundary for fixture-conformant inputs.

Compilation does not consume gas. Execution starts at `0` gas.

If `gasLimit >= 0` and consumed gas becomes greater than the limit, execution MUST fail with gas exhaustion.

### 12.2 Default gas schedule (normative)

The default gas schedule is:

| Field | Default |
|---|---:|
| `expressionBase` | 1 |
| `statementBase` | 1 |
| `documentRead` | 2 |
| `eventRead` | 1 |
| `stepsRead` | 1 |
| `currentContractRead` | 1 |
| `varRead` | 1 |
| `resultValueRead` | 2 |
| `pointerGetBase` | 1 |
| `pointerSetBase` | 3 |
| `objectSetBase` | 2 |
| `appendChangeBase` | 5 |
| `appendEventBase` | 5 |
| `forEachItem` | 1 |
| `functionCall` | 2 |

Every function invocation charges `functionCall`, including the root function. Therefore a trivial expression program costs at least `functionCall + expressionBase`.

### 12.3 Base charges (normative)

- Every evaluated expression charges `expressionBase`.
- Every executed statement charges `statementBase`.
- Every function invocation charges `functionCall`.
- Source wrappers and non-executed branches consume no gas.

### 12.4 Read charges (normative)

Read operators charge expression gas plus their read-specific charge:

| Operator | Additional charge |
|---|---:|
| `$document` | `documentRead` |
| `$event` | `eventRead` |
| `$steps` | `stepsRead` |
| `$currentContract` | `currentContractRead` |
| `$binding` | `varRead` |
| `$var` | `varRead` |
| `$resultValue` | `resultValueRead` |

`$const` does not add a read charge beyond expression gas.

Path-aware `$var` and `$const` object forms do not add a separate read charge for static paths. Dynamic `path` expressions consume their normal expression gas before the value-local pointer read.

`$kind` and `$isKind` charge only their normal expression tree costs. `$isKind.kind` is static authored data and consumes no expression gas.

### 12.5 Pointer and object update charges (normative)

`$pointerGet` charges:

```text
expressionBase + gas(object expression) + pointerGetBase + numberOfPathSegments
+ gas(default expression only when default is used)
```

`$pointerSet` charges:

```text
expressionBase + gas(value expression unless op is remove)
+ gas(object expression) + pointerSetBase + numberOfPathSegments + estimatedSize(value)
```

For `remove`, the value expression is not evaluated and value size is `0`.

`$objectSet` charges:

```text
expressionBase + gas(value expression) + objectSetBase + estimatedSize(value)
+ gas(object expression)
```

### 12.6 Append charges (normative)

`$appendChange` charges:

```text
statementBase + gas(value expression for add/replace) + appendChangeBase + estimatedSize(value)
```

For `remove`, the value expression is not evaluated and value size is `0`.

`$appendChanges` charges statement gas, gas for the list expression, and for each patch:

```text
appendChangeBase + estimatedSize(value)
```

`$appendEvent` charges:

```text
statementBase + gas(event expression) + appendEventBase + estimatedSize(event)
```

`$appendEvents` charges statement gas, gas for the list expression, and for each event:

```text
appendEventBase + estimatedSize(event)
```

### 12.7 Control-flow charges (normative)

`$if` charges statement gas and condition expression gas, then only the selected branch.

`$forEach` charges statement gas, input expression gas, `forEachItem` per iteration, and body gas.

`$returnIf` charges statement gas and condition expression gas. Its `expr` operand is evaluated only when the condition is truthy.

`$failIf` charges statement gas and condition expression gas. Its `message` operand is evaluated only when the condition is truthy.

`$let.vars` charges statement gas. In unordered form, all binding expressions are evaluated before any slot is assigned. In ordered form, each binding expression is evaluated and assigned in the explicit order.

`$and`, `$or`, `$coalesce`, `$some`, `$find`, `$findEntry`, and `$includes` short-circuit. Unevaluated operands or collection items consume no gas.

Collection expressions `$map`, `$filter`, `$flatMap`, `$reduce`, `$some`, `$find`, and `$findEntry` charge expression gas, input expression gas, `forEachItem` for each evaluated item, and body-expression gas for each evaluated item. `$reduce` also charges the initializer expression once after input evaluation.

`$objectFromEntries` charges expression gas, entries expression gas, and `forEachItem` for each input entry.

`$includes` charges expression gas, list expression gas, value expression gas, and `forEachItem` for each compared list item until it short-circuits or completes.

`$hasKey` charges expression gas, object expression gas, and key-expression gas only when the key is dynamic.

`$intrinsic` charges expression gas, gas for each evaluated payload field expression, and any gas charged explicitly by the registered intrinsic processor. `$intrinsic.type` is static authored Blue data and consumes no expression gas.

### 12.8 Size estimator (normative)

The size estimator is deterministic:

| Value | Estimated size |
|---|---:|
| `undefined` | 0 |
| `null` | 0 |
| scalar | `max(1, UTF-16 code-unit count of the BEX scalar text representation)` |
| list | list length plus sum of item sizes |
| object | number of keys plus sum of key UTF-16 code-unit lengths plus sum of value sizes |

The UTF-16 code-unit length rule is a BEX gas-accounting rule. It is not Blue Text schema length semantics.

An implementation MAY cache size estimates, but caching MUST NOT change gas results.

### 12.9 Known gas boundary (informative)

Pure expressions that return large values are not size-charged merely because they return a large value. Size charging occurs when values are inserted into output/update operations such as append changes, append events, `$objectSet`, and `$pointerSet`.

---

## 13. Errors and Diagnostics

### 13.1 Error classes (normative)

BEX distinguishes at least these error classes for fixtures and host APIs:

| Class | Meaning |
|---|---|
| compile error | Source is syntactically or statically invalid as BEX. |
| runtime error | Execution fails after successful compilation. |
| parse error | Source text cannot be parsed as Blue/YAML/JSON input. |
| output-conversion error | A BEX result cannot be converted to a Blue node. |
| gas exhaustion | Runtime failure caused by gas limit. |

A fixture implementation MUST classify errors deterministically.

### 13.1.1 Diagnostic fields (normative)

Fixture and host diagnostics MUST expose at least:

- error class;
- message;
- source path when known;
- operator name when known;
- function name or call-frame path when known;
- pointer or path operand when the failure is pointer-related and safely reportable.

Exact message text is not normative unless a fixture asserts `errorContains`.

### 13.2 Runtime failures (normative)

Runtime failure MUST stop execution. Accumulated changes and events are not committed by BEX. The host decides whether failed partial accumulators are observable for diagnostics.

Runtime failures include:

- invalid dynamic pointer, key, op, binding name, or step name;
- invalid type conversion;
- non-list input to list-only operators;
- non-object input to object-only operators where conversion is not defined;
- non-exact integer arithmetic;
- division by zero;
- function argument mismatch;
- invalid patch or event append;
- `$fail`;
- `$failIf` when its condition is truthy;
- invalid collection input or invalid collection result shape;
- invalid `$objectFromEntries` entries or keys;
- unsupported or failing intrinsic processor invocation.

### 13.3 Metrics (normative)

An execution implementation SHOULD expose deterministic metrics. The Java-derived metric categories include:

- expression evaluations;
- statement executions;
- function calls;
- document reads;
- result-value reads;
- patch appends;
- event appends.

Metrics are diagnostic and do not affect BEX semantics except where fixtures assert them under a declared profile.

---

## 14. Determinism and Security

### 14.1 Determinism (normative)

For a fixed:

- BEX source;
- optional definition node;
- execution context;
- immutable document views;
- bindings, event, steps, and current contract values;
- gas schedule and gas limit;
- Blue type matcher and Blue Language profile;
- registered intrinsic processor set and deterministic intrinsic processor behavior;

BEX execution MUST be deterministic.

BEX execution is synchronous from the program's point of view. Hosts MAY run BEX inside asynchronous runtimes, but BEX exposes no async primitives and no BEX operator may observe scheduling, timing, or interleaving.

### 14.2 No implicit authority (normative)

Except for explicitly registered `$intrinsic` processors, BEX has no implicit authority to:

- mutate host documents;
- apply changesets;
- emit events externally;
- read clocks or randomness;
- access files, networks, databases, or environment variables;
- execute Blue contracts.

All host data visible to BEX MUST be explicitly supplied through the execution context or through an explicitly registered intrinsic processor. A portable intrinsic processor MUST define deterministic semantics, failure behavior, and gas accounting for fixed inputs.

### 14.3 Host validation (normative)

A host MUST validate BEX output under its own authorization and content rules before applying changes or emitting events.

BEX patch accumulation is not an authorization decision.

### 14.4 Resource limits (normative)

Hosts SHOULD enforce gas limits and MAY impose additional deterministic limits on:

- source size;
- compile depth;
- expression nesting;
- statement count;
- list and object sizes;
- output size;
- pointer depth.

If a limit affects execution, failure MUST be deterministic.

---

## 15. Operator Reference Summary

This section is normative unless otherwise marked.

### 15.1 Expression operators

```text
Reading:
  $document, $binding, $event, $steps, $currentContract,
  $var, $const, $get, $changeset, $events, $resultValue

Type and conversion:
  $unwrap, $is, $kind, $isKind,
  $text, $integer, $number, $boolean, $object, $list

Strings:
  $concat, $pointerJoin, $join, $split, $startsWith, $sliceAfter

Logic and comparison:
  $eq, $ne, $gt, $gte, $lt, $lte,
  $and, $or, $not, $truthy, $empty, $isEmpty,
  $exists, $coalesce, $default

Numeric:
  $add, $subtract, $multiply, $divide

Objects and lists:
  $keys, $entries, $size, $listGet, $listConcat,
  $merge, $objectSet, $pointerGet, $pointerSet,
  $map, $filter, $flatMap, $reduce,
  $some, $find, $findEntry,
  $includes, $hasKey, $objectFromEntries

Control and host boundary:
  $choose, $call, $intrinsic, $literal,
  $null, $emptyObject, $emptyList
```

### 15.2 Statement operators

```text
$let, $set, $if, $forEach,
$appendChange, $appendChanges,
$appendEvent, $appendEvents,
$call, $return, $returnIf, $fail, $failIf
```

### 15.3 Operand naming guidance (informative)

Operator bodies SHOULD use BEX operand names rather than Blue wrapper names where possible. For example:

```yaml
# Preferred
$join:
  list: [a, b]
  separator: ","

# Avoid using Blue payload keys as BEX operand names
# unless the operator explicitly defines them.
```

BEX operator operands commonly use:

```text
node, list, input, pattern, object, key, path, val,
cond, then, else, name, expr, where, in, item, index,
acc, init, order, vars, op, args, function, message
```

### 15.4 Future standard library candidates (informative)

The following names are candidates for later BEX revisions and are not normative BEX 1.0 operators:

```text
$length, $slice, $findIndex, $findKey, $every,
$pick, $omit, $entriesMap,
$range, $min, $max, $sort, $match, $break, $continue
```

In BEX 1.0, `$size` is a collection/cardinality helper. For scalar text it returns `1`, not text length. A dedicated text code-point length operator is deferred.

---

## 16. Fixtures

### 16.1 Rich fixture root (normative for Fixture Profile)

A rich fixture document has:

| Field | Required? | Meaning |
|---|---:|---|
| `fixtureId` | yes | Stable fixture identifier. |
| `title` | yes | Human-readable title. |
| `targetStatus` | no | Stability or expected status marker. |
| `tags` | no | Fixture tags. |
| `context` | no | Execution context. |
| `blueDefinitions` | no | Map of custom BlueIds to source documents. |
| `gasSchedule` | no | Gas schedule overrides. |
| `programSource` | yes | BEX program source. |
| `expectation` | yes | Expected result or error. |

### 16.2 Fixture context (normative for Fixture Profile)

Fixture `context` MAY include:

```text
documentScope, rootDocumentSource, eventSource,
currentContractSource, stepsBinding, gasLimit, bindings
```

### 16.3 Fixture expectations (normative for Fixture Profile)

Valid expectation outcomes are:

```text
success,
compile-error,
runtime-error,
parse-error,
output-conversion-error,
parse-error-or-output-conversion-error,
gas-property
```

A success expectation MAY assert:

```text
resultSimple, changeset, events, gasUsed
```

An error expectation SHOULD assert an `errorContains` substring.

A gas-property expectation asserts a relation such as large output costing more than small output.

### 16.4 Manifest (normative for Fixture Profile)

A BEX 1.0 conformance suite MUST publish machine-readable fixtures with exact expected results, including gas where asserted.

The canonical fixture package is part of the BEX 1.0 release artifact and is versioned with this specification.

The BEX 1.0 release authority MUST publish the fixture package identity, either as a BlueId or as a content-addressed release artifact digest.

No inline reference gas totals or hashes need to live in this prose specification. Exact results live in the canonical fixture package.

Fixture manifests MUST list fixture files in deterministic order.

Implementations MUST run fixtures independently so fixture state cannot leak between tests.

### 16.5 Current fixture coverage (informative)

The BEX 1.0 fixture package is expected to cover the full normative operator surface listed in §15. The current package includes:

- core expression, statement, function, type-match, pointer, changeset, event, and output-boundary cases;
- ergonomics cases for `$returnIf`, `$failIf`, `$kind`, `$isKind`, path-aware `$var` / `$const`, and multi-bind `$let`;
- collection query cases for `$map`, `$filter`, `$flatMap`, `$reduce`, `$some`, `$find`, `$findEntry`, `$includes`, `$hasKey`, and `$objectFromEntries`;
- gas cases for default and overridden gas schedule fields, short-circuiting, collection iteration, guards, and output-size effects.

The machine-readable manifest, not this prose section, is authoritative for exact fixture file names, counts, package identity, and expected results.

### 16.6 Operator sufficiency corpus (informative)

The rich fixture suite verifies exact BEX semantics. Implementations and spec editors SHOULD also maintain a non-normative sufficiency corpus that translates representative higher-level workflows into BEX to test whether the current operator set is sufficient before adding more operators.

A useful corpus shape is:

| Source family | Suggested count | Purpose |
|---|---:|---|
| Kyverno validate-style policies | 30 | Guard validation, shape checks, list/object queries, early invalid results. |
| Kyverno mutate/generate-style policies | 20 | Deterministic patch and generated-object construction. |
| JMESPath/JSONata-style queries and transforms | 20 | Projection, selection, flattening, aggregation, object reconstruction. |
| JSON Patch emission edge cases | 10 | Pointer escaping, duplicate patch order, root and relative paths, lazy remove values. |

The sufficiency corpus is not a substitute for normative fixtures. Its role is to prevent operator creep: if a future workflow is hard to express, add or translate the case first, then decide whether the missing piece is a primitive, a documentation pattern, or protocol-level logic that belongs outside BEX.

---

## 17. Conformance Vectors

Conformance vectors are behavior-defining. A profile implementation MUST pass all vectors for its declared profile.

### 17.1 Compiler Profile vectors

- **C1.** An object with exactly one `$` key is an expression operator.
- **C2.** An object with multiple keys, including a `$` key, is a literal/computed object expression.
- **C3.** A statement object with multiple operator keys is rejected.
- **C4.** An unknown single expression operator is rejected.
- **C5.** `$literal` preserves nested unknown operators as data.
- **C6.** `$literal` does not permit BEX inside static Blue type or schema fields.
- **C7.** `$const` referencing an unknown constant is rejected at compile time.
- **C8.** `$call` referencing an unknown function is rejected at compile time.
- **C9.** Function calls with extra arguments are rejected.
- **C10.** Function calls with missing arguments are rejected.
- **C11.** Reserved Blue/BEX names in `args`, `constants`, `functions`, or `$call.args` are rejected.
- **C12.** Dynamic or computed `$is.pattern` is rejected.
- **C13.** Direct and indirect recursive function call graphs are rejected.
- **C14.** An `entry` function with arguments is rejected.
- **C15.** `$set` of an undeclared local variable is rejected.
- **C16.** Compile errors expose error class and include source path and operator name when known.
- **C17.** Null or empty statement items are rejected under strict BEX 1.0.
- **C18.** `$var` and `$const` object forms use static names and optional value-local paths; slash-containing names are not split as shorthand paths.
- **C19.** `$let.order` must list every `$let.vars` binding exactly once.
- **C20.** `$returnIf.value` is rejected; `$returnIf.expr` is the return payload field.
- **C21.** `$intrinsic.type` must be static and must resolve to a registered BlueId; unsupported BlueIds are compile errors.

### 17.2 Execution Profile vectors

- **E1.** `$document` resolves relative paths against document scope and absolute paths against the root.
- **E2.** `$event`, `$currentContract`, `$steps`, `$binding`, `$pointerGet`, and `$pointerSet` use value-local pointers.
- **E3.** Dynamic pointer expressions evaluating to `undefined` or `null` fail.
- **E4.** Dynamic text operands for keys, operation names, binding names, and step names reject `undefined` and `null`.
- **E5.** `$pointerJoin` escapes `~` and `/` correctly.
- **E6.** `$exists` is false only for `undefined`; it is true for `null`, `false`, empty text, and empty containers.
- **E7.** Numeric zero is truthy; empty object and empty list are falsy.
- **E8.** `$and`, `$or`, and `$coalesce` short-circuit and do not charge unevaluated operands.
- **E9.** `$integer` accepts large canonical integer text and rejects non-integral decimal text.
- **E10.** `$divide` rejects non-exact integer division.
- **E11.** `$split` preserves trailing empty parts with omitted or `-1` limit, and a positive limit returns at most that many parts with the final part containing the unsplit remainder.
- **E12.** `$keys` and `$entries` return object keys in lexicographic order over Unicode code points.
- **E13.** `$objectSet` with `undefined` value removes a key.
- **E14.** `$pointerSet` creates missing intermediate objects and rejects scalar intermediates.
- **E15.** Function calls use separate frames; callee locals do not leak to callers.
- **E16.** Function argument type mismatch is a runtime error.
- **E17.** `$is` propagates failure from evaluating `node`; successful non-convertible or non-matching values return `false`.
- **E18.** Object key exposure order is lexicographic for `$keys`, `$entries`, and `$forEach`.
- **E19.** BEX numeric equality is distinct from Blue scalar identity; BEX `1` and `1.0` compare equal as numbers.
- **E20.** Operand evaluation order is deterministic; `$call.args` expressions are evaluated by lexicographic argument name, and skipped lazy operands produce no side effects or gas.
- **E21.** Scalar text rendering is deterministic for booleans, integers, decimals, empty text, and non-ASCII text.
- **E22.** Object key ordering is independent of host map order and host locale; keys `a`, `U+E000`, and `U+1F600` are exposed in that Unicode code-point order, even on UTF-16 hosts.
- **E23.** `$kind` and `$isKind` report visible BEX runtime kinds, not implementation wrapper classes.
- **E24.** `$map`, `$filter`, `$flatMap`, `$reduce`, `$some`, `$find`, and `$findEntry` iterate lists in index order and objects in sorted-key order.
- **E25.** Collection query bindings restore any previous outer variable values after evaluation.
- **E26.** `$some`, `$find`, `$findEntry`, and `$includes` short-circuit and avoid later failures and gas.
- **E27.** `$objectFromEntries` rejects null or undefined keys and treats undefined values as key omission/removal.
- **E28.** `$hasKey` returns false for non-object input and true only for non-undefined object keys.

### 17.3 Statement and accumulator vectors

- **S1.** `$if` executes only the selected branch.
- **S2.** `$forEach` over a list binds item and index correctly.
- **S3.** `$forEach` over an object visits keys in lexicographic order over Unicode code points.
- **S4.** `$forEach` duplicate binding names are rejected.
- **S5.** `$appendChange` accepts `add`, `replace`, and `remove` only.
- **S6.** `$appendChange` and `$appendChanges` require `val` for `add` and `replace`.
- **S7.** `remove` patches do not evaluate `val`.
- **S8.** Duplicate patch paths are preserved in append order.
- **S9.** `$appendEvent` and `$appendEvents` reject `undefined` events.
- **S10.** `$resultValue` sees the latest accumulated patch for an exact path.
- **S11.** `$resultValue` parent reads reflect descendant patches.
- **S12.** `$resultValue` root replacement and root removal have deterministic overlay behavior.
- **S13.** `$resultValue` preserves duplicate patch paths and applies them in append order.
- **S14.** `$return` exits the current function, not merely the innermost loop.
- **S15.** `$resultValue` list index removal is non-shifting with later indexes retaining their positions.
- **S16.** `$resultValue` applies parent replacement followed by child replacement in order.
- **S17.** `$resultValue` list-index removal creates a sparse non-shifting overlay slot; the removed index reads as `undefined`, later indexes retain positions, and `$size` reports positional extent.
- **S18.** A statement function with no `do` completes normally and returns the default result value.
- **S19.** Root-level `$return` and `$returnIf` exit the root function and stop later root statements.
- **S20.** `$returnIf` and `$failIf` evaluate their payload/message only when their condition is truthy.
- **S21.** Unordered `$let.vars` is parallel; ordered `$let.vars` is sequential according to `order`.

### 17.4 Host Boundary vectors

- **H1.** Converting `undefined` to a root Blue node fails.
- **H2.** Converting a list containing `undefined` fails.
- **H3.** Converting an object with `blueId` and sibling fields fails.
- **H4.** Converting an object with `properties` as a Blue wrapper fails.
- **H5.** Converting a node with mixed payload kinds fails.
- **H6.** Strict BEX 1.0 output conversion rejects `constraints` and non-Blue-Language schema keys; a declared Java Compatibility Output Extension Profile may accept documented compatibility schema keys without claiming portable Blue Language 1.0 output.
- **H7.** Output conversion rejects unsupported list-control fields.
- **H8.** Computed Blue language fields such as `valueType` are preserved as Blue language fields, not ordinary child fields.
- **H9.** BEX numeric output conversion to Blue preserves or selects the target Blue numeric type deterministically and rejects invalid finite-Double conversions.
- **H10.** Converting a sparse overlay list with an undefined removed slot to Blue output fails under the Host Boundary Profile.
- **H11.** Strict output conversion rejects computed `blue`; an authoring-output extension profile must be declared to emit Blue Source Document preprocessing directives.
- **H12.** Blue `Double` read into BEX is independent of source token spelling, and BEX decimal output to Blue `Double` follows the declared deterministic conversion and rounding rule.

### 17.5 Gas vectors

- **G1.** A root expression program charges one root function call plus expression gas.
- **G2.** Gas exhaustion occurs when used gas becomes greater than a non-negative limit.
- **G3.** `$and`, `$or`, and `$coalesce` do not charge skipped operands.
- **G4.** `$if` does not charge the unselected branch.
- **G5.** `$forEach` charges `forEachItem` per iteration.
- **G6.** `$pointerGet` charges per path segment.
- **G7.** `$pointerSet`, `$objectSet`, `$appendChange`, and `$appendEvent` charge estimated value size.
- **G8.** Large appended or inserted values cost more gas than small values under the default schedule.
- **G9.** `$binding` charges expression gas plus `varRead`.
- **G10.** Size estimation for non-ASCII scalar text and non-ASCII object keys uses the specified length unit, independent of host string internals.
- **G11.** Collection expressions charge `forEachItem` only for evaluated items; short-circuited items consume no gas.
- **G12.** `$returnIf`, `$failIf`, and `$let.vars` charge statement gas according to their selected/evaluated operands.
- **G13.** `$intrinsic` gas includes evaluated payload expressions and processor-charged gas.

---

## 18. Worked Examples

### 18.1 Simple expression

```yaml
expr:
  $add:
    - 2
    - 3
```

The program returns integer `5`.

### 18.2 Statement program with changes and events

```yaml
do:
  - $appendChange:
      op: replace
      path: /status
      val: CONFIRMED
  - $appendEvent:
      type: StatusChanged
      status: CONFIRMED
  - $return:
      changeset:
        $changeset: true
      events:
        $events: true
```

The program accumulates one patch and one event, then returns them as data.

### 18.3 Constants

```yaml
constants:
  threshold: 400
expr:
  $gte:
    - $document: /amount
    - $const: threshold
```

A missing `threshold` constant would be a compile-time error.

### 18.4 Function with static argument pattern

```yaml
functions:
  isLarge:
    args:
      amount:
        type: Integer
    expr:
      $gte:
        - $var: amount
        - 400
expr:
  $call:
    function: isLarge
    args:
      amount:
        $document: /amount
```

The argument is evaluated and then matched against the static Blue pattern before the function body runs.

### 18.5 Reading host bindings

```yaml
expr:
  $binding: actor/id
```

This reads binding `actor` at value-local path `/id`.

Equivalent object form:

```yaml
expr:
  $binding:
    name: actor
    path: /id
```

### 18.6 Dynamic pointer construction

```yaml
expr:
  $document:
    $pointerJoin:
      - reservations
      - $event: /reservationId
      - status
```

If `reservationId` contains `/` or `~`, `$pointerJoin` escapes it correctly.

### 18.7 `$resultValue`

```yaml
do:
  - $appendChange:
      op: replace
      path: /status
      val: CONFIRMED
  - $return:
      statusAfterPatch:
        $resultValue: /status
```

The returned `statusAfterPatch` is `CONFIRMED`, even though the host document has not been mutated by BEX execution.

### 18.8 `$resultValue` non-shifting list removal

```yaml
context:
  rootDocumentSource:
    items: [A, B, C, D]
programSource:
  do:
    - $appendChange:
        op: remove
        path: /items/1
    - $return:
        removedExists:
          $exists:
            $resultValue: /items/1
        stillAt2:
          $resultValue: /items/2
        positionalSize:
          $size:
            $resultValue: /items
```

`removedExists` is `false`. `/items/2` still reads `C`. `positionalSize` is `4`. The overlay does not shift later indexes. Returning the whole sparse overlay list directly would fail Blue output conversion under the Host Boundary Profile.

### 18.9 `$resultValue` parent and child replacement

```yaml
context:
  rootDocumentSource:
    order:
      status: DRAFT
      total: 10
programSource:
  do:
    - $appendChange:
        op: replace
        path: /order
        val:
          status: CONFIRMED
    - $appendChange:
        op: replace
        path: /order/total
        val: 20
    - $return:
        orderAfter:
          $resultValue: /order
```

The returned `orderAfter` is:

```yaml
status: CONFIRMED
total: 20
```

Patches are applied in append order, so the child replacement is applied after the parent replacement.

### 18.10 Guard-style return

```yaml
do:
  - $returnIf:
      cond:
        $not:
          $isKind:
            val:
              $event: /message/request/summary
            kind: text
      expr:
        changeset: []
        events:
          - type: Conversation/Proposed Change Invalid
            reason: summary is missing
  - $return:
      accepted: true
```

If the summary is missing or not text, the root function returns the invalid result and the later `$return` is not executed.

### 18.11 Collection projection

```yaml
expr:
  $map:
    in:
      $event: /message/request/changeset
    item: patch
    expr:
      op:
        $var:
          name: patch
          path: /op
      path:
        $var:
          name: patch
          path: /path
```

The expression projects each requested patch into a normalized patch-like object without mutable accumulator boilerplate.

### 18.12 Intrinsic signature verification

```yaml
expr:
  $intrinsic:
    type:
      blueId: CommonCryptoEd25519Verify
    publicKey:
      $const: trustedSignerPublicKey
    message:
      $event: /message/canonicalBytes
    signature:
      $event: /message/signature
```

The host must have a processor registered for the BlueId of `CommonCryptoEd25519Verify`. The intrinsic definition must specify the exact message bytes being verified.

### 18.13 Literal escape

```yaml
expr:
  $literal:
    $call:
      function: notExecuted
      args: {}
```

The result is an object containing the `$call` key as data. No function is invoked.

---

## Appendix A — BEX Value and Blue Node Boundary

This appendix is informative.

BEX values are optimized for execution, not identity. Blue nodes are optimized for content semantics and BlueId. A host should not assume that every BEX object is already a valid Blue node. The conversion boundary is deliberately strict so that invalid Blue output fails before it is stored or hashed.

Examples of invalid output:

```yaml
# Invalid: mixed pure reference and content
blueId: X
name: Something
```

```yaml
# Invalid: Blue has no properties wrapper
properties:
  a: 1
```

```yaml
# Invalid: mixed payload kinds
value: 1
items: [2]
```

---

## Appendix B — Gas Calculation Examples

This appendix is informative.

A trivial root expression:

```yaml
expr: 1
```

uses:

```text
functionCall + expressionBase = 2 + 1 = 3
```

A short-circuit expression:

```yaml
expr:
  $and:
    - false
    - $fail: should not run
```

charges for the root function, the `$and` expression, and the first operand. The `$fail` expression is not evaluated.

Size estimation examples under the default BEX gas-accounting length unit:

```text
"hello" => 5
"" => 1
true => 4
12345 => 5
"😀" => 2
```

The `"😀"` example counts UTF-16 code units for gas accounting. This is not Blue Text `minLength` or `maxLength` semantics.

---

## Appendix C — Common Implementer Mistakes

This appendix is informative.

### C.1 Do not treat every `$` key as an operator

Only an expression object with exactly one `$` key is an expression operator. Multi-field dollar objects are data.

### C.2 Do not allow executable BEX in Blue type fields

`type`, `itemType`, `keyType`, `valueType`, `blue`, and `schema` are static Blue definition positions.

### C.3 Do not use Blue reserved keys as user argument names

Function argument names and call argument names live in plain name containers. Reserved Blue keys such as `value`, `items`, `type`, and `schema` are invalid there.

### C.4 Do not apply changesets automatically

BEX accumulates patches. The host applies or rejects them.

### C.5 Do not delete or reorder patches

Patch order and duplicate patch paths are meaningful.

### C.6 Do not interpret dynamic null pointers as root

Omitted static paths can mean root/default. Dynamic `null` or `undefined` paths are errors.

### C.7 Do not forget root function gas

Every execution invokes the root function and charges `functionCall`.

### C.8 Do not expose mutable host nodes

Host nodes visible to BEX must be immutable for the duration of execution or snapshotted.

### C.9 Do not treat BEX numeric equality as Blue identity

BEX `1` and `1.0` compare equal as execution numbers. Blue scalar identity still distinguishes Blue `Integer` from Blue `Double`.

### C.10 Do not expose object keys in host map iteration order

BEX object key exposure order is lexicographic by Unicode code point, independent of host map insertion or iteration order.

### C.11 Do not swallow runtime failures inside `$is.node`

If evaluating `$is.node` fails, that runtime failure propagates. Only successful non-conversion or non-match returns `false`.

### C.12 Do not treat `$size` as text length

`$size` is a collection/cardinality helper. For scalar text it returns `1`.

### C.13 Do not implement `$binding` with host-defined gas under the Fixture Profile

Fixture-conformant `$binding` reads charge expression gas plus `varRead`.

### C.14 Do not rely on host operand evaluation order

BEX defines operand evaluation order. Host map order and source parser map order are not execution semantics.

### C.15 Do not assume `$split` uses regex semantics or drops trailing empty parts

`$split` uses literal separators. Omitted or `-1` limit preserves trailing empty parts.

### C.16 Do not materialize sparse `$resultValue` list overlays as dense Blue lists

Non-shifting list removals create sparse overlay slots. Converting such an overlay list to Blue output fails while a removed slot is `undefined`.

### C.17 Do not emit `blue` under the strict Host Boundary Profile

Computed `blue` requires an explicit authoring-output extension profile.

### C.18 Do not confuse BEX gas string length with Blue Text schema length

BEX gas size estimation counts UTF-16 code units. Blue Text schema length semantics are defined by the Blue Language profile.

### C.19 Do not rely on Java compatibility schema keys without declaring the profile

Output accepted only by the Java Compatibility Output Extension Profile is not portable Blue Language 1.0 output.

### C.20 Do not use collection queries as leaking local scopes

`$map`, `$filter`, `$flatMap`, `$reduce`, `$some`, `$find`, and `$findEntry` temporarily bind item/key/index/acc slots. Implementations MUST restore previous local values after the expression.

### C.21 Do not split `$var` or `$const` names on `/`

Path-aware reads use object form with `name` and `path`. The scalar form is a variable or constant name, even when it contains `/`.

### C.22 Do not implement `$intrinsic` as arbitrary host callback dispatch

`$intrinsic.type` is static Blue data. The operation is allowed only when the active registry has a processor for the resolved BlueId. Unsupported BlueIds fail at compile time.

---

*End of Blue BEX Specification 1.0.*
