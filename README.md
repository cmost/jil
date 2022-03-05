# JSON Intermediate Language

The following is a specification for a JSON Intermediate Langauge.

This specification provides a standard way to serialize (and evaluate) expressions using JSON.

## Goals

The primary goal of this specification is to provide a portable way to define expressions for
use in a variety of places:

* For dynamic configuration (such as conditional rollout of features or environment-specific configuration).
* For dynamic document formats (rich data with tables/grids/views etc).
* For rich queries (much like LINQ or Underscore.js but meant for broad scenarios).
* For a rich modern packaging system (as the langauge that defines packages and side-effects).
* For a rich algorithm language that can be transpiled to various languages.

In all these cases, the langauge should be efficiently embeddable in C, Go, Rust, Python and JS.

## File Extensions

All JIL files MUST use the `.jil` extension.

## Media/MIME Type

The media type for JIL is tentatively `application/vnd.jil+json`

## Atomic values

JIL natively supports only three types of literals:

* integers (32-bit signed: -2,147,483,648 to 2,147,483,647)
* boolean (true/false)
* string (regular JSON string values)

All other literal values in JIL are expressed via function calls to the constructors. For example, a
currency value may be represented as `dollars.new(53)` (encoded appropriately into JIL).

All strings in JIL MUST be unicode, encoded into NFD form as defined in https://unicode.org/reports/tr15/.

## Operator expressions

JIL allows simple operators to be used in expressions.  For example, `1 + 2 + 3` would
be encoded in JIL like so:

```json
["+", 1, 2, 3]
```

Operator expressions have the form: `["op", operands...]`.  The following are the operators in JIL:

* Arithmetic: `+`, `-`, `*`, `/`
* Inequality and Equality: `<`, `<=`, `>`, `>=`, `==`, `!=`
* Logical: `&`, `|` and `!`
* Field Access: `.`

Arity of operators:

* `-` MUST have one or more operands.
* '!' MUST have exactly one operand.
* All other operators MUST have two or more operands.

All operators with multiple operands are left-associative. The comparision operators are chained with pair-wise
comparisons that are all ANDed together. For example: `x < y < z` is evaluated as `x < y & y < z`.

Note that arguments can be any valid JIL expression.  For example, `2*3 + 1` would be encoded
into JIL like so:

```json
["+", ["*", 2,3], 1]
```

## Operator Behaviors

All arithmetic operators MUST use 32-bit arithmetic.  Overflows should retain the least significant
bits as well as the sign.  Division MUST return the integer part of the result. Arithmetic on
invalid types should return a type error.  If any of the underlying values are already error values,
the underlying error value MUST be wrapped.

Floating point arithmethic or arithemtic on other types are not defined but may be provided
via specific extensions.

Logical operators are only defined on boolean types.  Boolean expressions with mismatched types MUST
return a type error EXCEPT when:

* one of the operands in an `|` operation evaluates to `true` -- in this case, the result is `true`.
* one of the operands in an `&` operation eavluates to `false` -- in this case, the result is `false`.

The `==` and `!=` operators MUST accept values of any types (including error values) and MUST be
defined and work consistently:  two values of different types MUST NOT be equal.  Two values of
the same type MUST return true for one of the two operators and false for the other.  String equality
is defined as comparision on the NFD form (since strings are required to be NFD to start with).

The other comparision operators `<`, `>`, `<=`, `>=` are defined on integers and boolean only.
For integers, standard arithmetic comparison behavior applies.  For boolean values, `false < true`.
When more than two operands are proided, the operation is seen as a chain: `x < y < z` is
treated as `x < y & y < z`.  If invalid types are used, the evaluation MUST return a type error
unless one of the binary comparisons evaluates to a `false` (in which case the result MUST be `false`).

## Identifiers

Identifiers in JIL are just simple single-element JSON arrays.  For example, `x` would
be encoded as `["x"]` in JIL.

There are no restrictions on what values identifiers can have.  In particular, identifiers
can use all unicode characters.

## Function Call Expressions

Simple function calls would be encoded into JIL using a simple array with the first
element representing the function while the rest of the elements represent arguments
to the function, much like in Lisp.  For example, `foo(x, y)` would be represented as:

```json
[["foo"] ["x"] ["y"]]
```

Note that the first element is actually an identifier `foo` and so the first element
of the call expression array is `["foo"]`.

Note also that any expression is allowed as the first element of the array.  For example,
`strings.length("hello")` would be encoded into JIL like so:

```json
[["." ["strings"] "length"] "foo"]
```

## Object Expressions

JIL natively supports a generic object with arbitrary key and value types.  These are
expressed via an object expression like so:

```json
[["object"], key1, value1, key2, value2, ....]
```

Note that both the keys and values may be arbitrary JIL expression. A simple JSON object
MUST be used for the case where all the keys are strings:

```json
{"key1": value1, "key2": value2, ...}
```

## The DOT operator

The dot operator extracts fields and can be chained.  For example, `x.y.z` would be
encoded into JIL as:

```json
[".", ["x"], "y", "z"]
```

Also, JIL allows arbitrary types for the DOT selector: `args.0.real` would be:

```json
[".", ["args"], 0, "real"]
```

## Conditional expressions

There are two conditional expressions in JIL: `if` and `when`:

```json
[["if"], condition, then, else]
[["when"], condition1, then1, condition2, then2, else]
```

In the forms above, both conditions and then/else clauses are arbitrary expressions. For the `when`
expression, the conditions are evaluated in order with the final `else` expression used if no
condition matches.  if the condition evaluates to a non-boolean value, it is treated as false.

## Error encodings

JIL encodes errors as just objects with a key named `error` and whose value MUST be a
string representing a human readable description of the error.  More error details may
be present in other fields of the object.  For example:

```json
{
   "error": "parse error",
   "file": "foo.jil",
   "line": 43,
   "column": 22
}
```

Note that the error expression MAY use the more general object form:

```json
[["object"], "error", "parse error", "format", "parse error in $1 at $2:$3", 1, "foo.jil",  2, 43, 3, 22]
```

In the example above, numeric keys are used (which are not allowed in JSON objects and so
the "object" form may be used.  The example above uses a `format` key -- this is just an
example.  JIL does not define any formatting of errors though extensions might define those.

## Wrapping errors

Errors can be wrapped.  In those case, the inner error is just defined via an "inner" key:

```json
{
   "error": "top level error",
   "inner": {
       "error": "inner error"
   }
}
```

Note that for an object to be considered an error the following MUST be true:

1. It MUST have a key named `error` whose value is a string value.
2. It either does not have an `inner` key at all or the value of the `inner` key is an error.

## Where Expression

JIL also supports a where expression (much like a `let` expression in LISP):

```json
[["where"], value, {name1: def1, name2: def2, ...}]
```

Here `value` represents the expression to evaluate.  The second argument represents a
set of names and associated definitions.  These names can be referred to in the main value
to evaluate and they stand in for the corresponding definition.  These names may also
appear in the defintions.

For example, consider:

```json
[
  ["where"],
  ["+",  ["x"], ["y"]],
  {
    "x": 42,
    "y": ["-", ["x"], 40]
  }
]
```

The value of that JIL expression is `x + y` (where `x` is defined as `42` and `y` is defined as `x - 40`).
So, the final result is `42 + 42 - 40` or `44`.

The where clause effectively introduces a scope where the names can be referred to within the
main value expression as well as other definitions.

Note that `where` clauses may be nested in which case the names are treated as lexically scoped
(i.e can only be referred to within the expressions in the definitions or value).

Other rules:

1. Names in the `where` clause should not shadow other names already in scope at that point. Lexical
scoping is used where the scope is defined on some parent scope.

2. The same name MUST not appear twice in the same `where` clause.

3. Circular references MUST not be present: if a defintion of name A references a name B, the definition
of name B cannot directly or indirectly reference name A.

In all these case, if the rules are violated, the where expression should result in a name validation
error.

## Functions

JIL allows functions to be defined via a simple `[["func"] value]` expression.  The value is the
definition of the function and it can refer to arguments via the `args` identifier.

For example:

```json

[
   [["func"], ["+", [".", ["args"], 0], [".", ["args"], 1]]],
   5,
   10
]
```

The expression above defines a function which adds the first argument (`[".", "args"], 0]`) and the
second (`[".", ["args"], 1]`) and then executes this function against arguments 5 and 10.

A slightly more readable version using the where expression would look like this:


```json
[
  ["where"],
  [["sum"], 5, 10],
  {"sum": [
    ["func"]
    [
      ["where"],
      ["+", ["first"], ["second"]],
      {
        "first": [".", ["args"], 0],
        "second": [".", ["args"], 1],
      }
   ]
  ]}
]
```

In pseudo-code, the above would map to something like:

```
sum[5, 10].where{
  sum: func((first + second).where{
    first: args.0
    second: args.1
  }
}
```

## Extensions

Extensions can be defined via an `using` expression:

```json
[
  [["using"] "uri"]
  expression
]
```

The `using` function takes one primary argument, which is expected to be an URI that defines an extension.
The specific mechanism by which the URI is interpreted is out of scope of this document.

The using function itself returns the extension which is then passed the body of the expression.  The
extension can modify the underlying JIL and rewrite it as well as inject variables in the scope. The
specific mechanisms will be defined in a separate spec.

## Standard attributes and methods

All values in JIL are expected to have the following attributes:

* failed -- true if the current value represents an error, false otherwise
* ok -- false if the current value represents an error, true otherwise

All values in JIL are expected to have the following methods:

* equals -- equivalent of the == operator
* lessThan -- a more generic version of the < operator that is guaranteed to work on any pair of values.
* less -- similar to lessThan
* moreThan -- inverse of less
* more -- inverse of lessThan
* is -- type checking.  See Types
* cast -- type casting.  See Types
* coerce -- type coercion.  See Types
* jil - convert the value to a valid JIL expression that is guaranteed to evaluate to the same value.

The `jil` method returns a JSON string which, if evaluated, MUST result in a value that is identical to
the current value when compared with the equals method or the equality operator. In addition the `jil`
method should return the canonical form of the expression.  For example, if an extension defines a
`Date` value via `Date.new("12/20/1994")`, the jil string should not encode the raw object but instead
should encode the call expression `[[".", ["Date"], "new"] "12/20/1994"]`.

## Types

JIL defines four types: `integer`, `string`, `boolean` and `error`.  Extensions may define other
custom types and types can also be defined programmatically in JIL itself.  A type is just an object
which has the following fields (it may have other fields):

1. new: this must be a function which generates values of this type.
2. is: this is a function which checks if the value belongs to this type.
3. cast: this is a function that attempts to modify the input value to this type.  It can return errors.
4. coerce: this is a function that alwasy returns a valid value of this type based on the input args.

For example, a `complex` type can be defined like so in psuedo-code:

```
{
  new: func[{real: r, im: i}.where{r: args.0, i: args.1}]
  is: func[(r.is[integer] & i.is[integer]).where{r: args.0.real, i: args.0.im}]
  cast: func[{real: args.0.cast(integer), im: 0}]
  coerce: func[{real:args.0.coerce(integer), im:0}]
}
```

When an expression like `value.is(type)` is evaluated, the `is` function defined by the type is used.
Similarly, `cast` and `coerce` methods on any value just proxy it to the corresponding functions
defined on the type.

Note that the pseudo-code above is just to illustrate the behavior -- it does not properly implement
the jil method which should ideally return the JIL JSON equivalent of `complex.new(x, y)` rathern than
`{"real": x, "im": y}`.

## Representing static data via type constructor

JIL can be used to serialize rich static types. For example, a complex value can be represented via:

```json
[["." ["complex"] ["new"]], 5, 10]
```

Or in pseudo code: `complex.new[5, 10]`.

Extensions may provide a set of standard types (such as Tables, Trees, etc) allowing the types to
be properly captured.

## Algorithm to evaluate JIL

1. If the value is a string, convert it to the Unicode NFD form and return it.
2. If the value is a boolean, return it.
3. If the value is a number and it is a 32-bit signed integer, return it.
4. If the value is a number but not a 32-bit signed integer, return an error.
4. If the value is an array with a single string element, lookup the value of this from the scope.
If it is found in the scope, return it.  Else return an error.
5. If the value is an array where the first element is an operator, evaluate the operation.
6. If the value is an array but either with many elements or with a non-string first element,
evaluate the first element.  If it evaluates to a non-function, return an error.  If it evaluates
to a function that accepts JIL expressions rather than values, pass the arguments as JIL expressions.
If it evaluates to a function that accepts values as arguments, evaluate the arguments and call that
function.  If it evaluates to a non-function, return error.
7. If the value is an object, return the object.

### Evaluating operators:

1. Regular arithmetic operators evaluate all the arguments and then proceed with arithmetic.
2. Comparision operators evaluate arguments on demand and do pair-wise comparison, short-circuiting
once the result is known.
3. Logical operators evaluate argumentes one at a time and short-circuit immediately.
4. Field access evaluates the first argument and then evaluates the field access one at a time. All the
built-in properties (like "failed", "ok", "cast", etc) are implemented directly by the operator. Array
indexing is also implemented directly by the operator.  For everything else, the underlying object is
treated like a dictionary with arbitrary types and used for lookup.

### Builtin functions that work on JIL expressions:

The builtin functions `if/when` should evaluate only the arguments necessary to short-circuit calculation.

The builtin function `where` first needs to validate that there is no duplicate name, invalid name or
recursive name in the current expression (but such errors in inner sub-expressions are ignored) before
injecting the definitions within the current scope and evaluating the main expression.

## Path for error reporting

When errors happen while evaluating an expression, the call stack cannot refer to line numbers. Instead
each line in the call stack would refer to the path within the JIL expression for where the error happens.

The path is jsut a sequence of keys/indices in the JIL expression.

