# Soli language reference (verified on soli 2.28.2; rechecked on 2.36.3)

Run anything below with `soli file.sl` or `soli -e '...'`. Scripts execute
top-to-bottom; no `main`.

## Variables and constants

```soli
name = "Alice"            # declares (no `let` needed)
let age: Int = 30         # `let` for a type annotation
const MAX_SIZE = 100
x = 5
x += 1                    # also -=, *=, /=, %=
x ||= 10                  # assign if falsy (nil, false, 0, "", [], {})
y = nil
y ??= 5                   # assign only if nil (keeps false / 0)
flag = false
flag ??= true             # stays false
h = {"a": 1}
h["b"] ||= 2              # works on hash entries
```

Inside a block (`each do`, `fn`), assigning to a name that exists outside
**mutates the outer variable** (closures capture by reference):

```soli
total_sum = 0
[1, 2, 3].each do |n|
  total_sum += n
end
total_sum   # 6
```

Use `let` inside the block only when you want a fresh local with the same name.
Avoid naming a variable after a builtin/global function (`url`, `response`,
`post` in specs…): `defined("url")` tells you whether a name is taken.

## Types and truthiness

`type(v)` / `v.class` return lowercase names: `int`, `float`, `string`,
`null`, `array`, `hash`; class instances report the class name (`Point`).

| Value | Truthy? |
|---|---|
| `false`, `nil`, `0`, `""`, `[]`, `{}` | falsy |
| `0.0`, `"a"`, `1`, `[0]` | truthy |

`nil == null` is `true`. Write `nil`.

**Static type checker.** `soli script.sl`, `soli -e` and `soli check` type-check
first. Any value is a condition (`if items.length`, `if hash` pass). An
unannotated local reassigned with another type widens (`total = 0` then
`total = total * 1.5` is a Float) — only an annotated binding (`let x: Int`, a
typed parameter) refuses a value of another type, and so does a bare
`helper = 5` where `helper` is a `def` of the file, or a builtin at the top
level (both engines rebind the function; `let helper = 5` shadows it). Inside
a def, method, lambda or block, `len = 3` is a plain local: the builtin stays. `T?` takes `nil`:

```soli
def label(count: Int?) -> String
  return "none" if count.nil?

  "#{count} item(s)"
end
label(nil)           # "none" — accepted by the checker and both engines
```

Unions: `def show(id: Int | String)`, `-> String | nil` (`| nil` = `?`),
`(Int | String)[]`. A value that fits no member is refused; a union value
only goes where every member fits; reading one gives `Any` (no narrowing).
Between a block's pipes the `|` closes them — write `|x: (Int | String)|`.
Both engines check union return types.

Generics: `class Box<T>` (brackets against the name; `class B<A` is still
inheritance), `def first<T>(xs: T[]) -> T`, `def map<U>(f: (T) -> U) ->
Box<U>`, annotations `Box<Int>`, `Hash<String, Int>`, `Array<Int>`. Erased at
run time; `soli check` infers `Box.new(41)` as `Box<Int>` and `first(["a"])`
as `String`, and checks what follows.

Every statement of a body is checked even after an error, so `soli check`
lists all of a file's errors. Skip the checker with `--no-type-check`.

`Klass.new(…)` is an instance of `Klass`: a member the class does not have is
an error (`Cannot access member 'totl' on type 'Pricing' (did you mean
'total'?)`). An `@x = …` in a method or the constructor makes `x` a field,
declared or not. Every `def` is checked for arity (defaults counted). In an
application, `soli check` knows the other auto-loaded files by their
declarations — a sibling's `def` arity, a sibling class's members, a parent
in another file — except `Model` fields, classes under `< Controller`, and a
name two files declare.

Levels: `# soli: off|loose|standard|strict` among a file's top comment lines
(scripts honour it; `off` = unchecked), else `soli check --level`, else
`[check] level` in `soli.toml`. `loose` reports only undefined names/types,
unknown members, non-callables and wrong arity; `strict` also requires every
parameter and return type (`E0404`; `&block` and constructors' return exempt,
a def returning nothing writes `-> Void`).

`soli check` also reads the views' code (`--no-views` skips): a misspelt
helper, a wrong argument count or an unknown member of a known class is
reported; `@x` and bare names a partial is passed are not. An undefined
`*_path`/`*_url` in an application is a route helper, never an error.

## Numbers

```soli
10 / 3          # 3   — Int / Int is integer division
10 % 3          # 1
9223372036854775807 + 1   # raises "integer overflow: 9223372036854775807 + 1 does not fit in an Int (range …)"
                          # += -= ++ -- raise the same way and leave the variable unchanged; they never wrap
7 / 2.0         # 3.5
7.to_f / 2      # 3.5
(-5).abs        # 5   — note: -3.abs parses as -(3.abs) = -3
16.sqrt         # 4   (floats with no fraction print without ".0")
2.pow(10)       # 1024
3.5.round       # 4
3.14159.round(2) # 3.14
5.round         # 5 — Int answers round/floor/ceil/truncate too
1250.round(-2)  # 1300
10.5.floor      # 10
10.2.ceil       # 11
5.between?(1, 10)  # true
0.zero?            # true — `User.count.zero?` reads better than `== 0`
5.positive?        # true; (-3).negative? true; 0 is neither
4.even?            # true; 7.odd? true (Int only — a Float has zero?/positive?/negative?)
7.clamp(1, 5)   # 5
1_000_000       # underscores allowed
int("42")       # 42
float("3.5")    # 3.5
str(5)          # "5"
```

There is no `Math` namespace, no `sqrt()`/`abs()`/`max()` globals, no
`format`/`sprintf`; `"..." % [...]` is not supported. Random: `Crypto.random_hex(n)`.

## Strings

```soli
greeting = "Hello #{name}!"          # only interpolation form; works in '...' too
"Value: " + 42                       # "Value: 42" (auto-converts)
"Name: #{person["name"]}"            # quotes inside #{} are fine
r"C:\Users\name"                     # raw, single line
q = """
  raw, multi-line: \n stays literal, "quotes" fine
"""
"a\tb\n\u00e9\u{1F600}"              # escapes: \n \t \r \\ \" \' \0 \e \u
buf = ""
buf << "ab"                          # append (plain variables only)
buf += "c"
```

Not valid: `\(name)` (lexer error), `@"..."` (parse error), `[[ … ]]` strings
(that is a nested array now; `soli fmt --migrate-raw-strings` rewrites old
code), `${}`. A string with its own `#{}` nested inside an interpolation
(`"outer #{"#{a}.#{b}"}"`) lexes since 2.35.0; up to 2.34.0
it failed with "Unterminated string" — build the inner string in a variable
there.

`"é".length` is **2** — length counts bytes. String methods: see
`stdlib-builtins.md`. But `s[i]` indexes by **character**: `"héllo"[1]` is
`"é"`, `"héllo"[-1]` is `"o"` (negative counts from the end, both engines).
Out of range raises `Index out of bounds: 5 (length 5)` (the length in
characters) instead of returning nil; write `s[i] rescue nil` for an "or nil"
read.

## Arrays

```soli
numbers = [1, 2, 3]
combined = [...a, ...b]     # spread; `a + b` and `a.concat(b)` also work
numbers[0]                  # 1
numbers[-1]                 # 3
%w[a b c]                   # ["a", "b", "c"]
[[1, 2], [3]]               # nested array (not a string)
```

`[1, 2, 3] - [2]` is `[1, 3]`: array difference drops every element equal to
one on the right (on both engines since the post-2.29.3 VM fixes; the VM
raised "Cannot subtract array from array" before).

## Hashes

```soli
person = {"name": "Alice", "age": 30}
h = {name: "Al", age: 3}    # bare keys become string keys: h["name"]
person.name                 # dot access (preferred) — nil if missing
row = {"size": 2048, "key": "abc"}
row.size                    # 2048 — a key named like a method is the key
row.size()                  # 2    — parentheses always call the method
row.key("abc")              # "key"
{"a": 1}.size               # 1    — no such key: the method answers
person["name"]
person.email = "a@b"        # dot assignment adds a key
person["city"] = "Paris"
```

Iterate with `h.each do |key, value| … end` or `for key in h.keys`.
**`for k, v in h` binds k = key and v = an index (0, 1, …)** — silently wrong.

## Control flow

```soli
if n < 0
  "neg"
elsif n == 0
  "zero"
else
  "pos"
end

print("adult") if age >= 18
print("minor") unless age >= 18
label = x > 2 ? "big" : "small"

while i < 3
  i += 1
end

for i in 0..3               # 0, 1, 2
  next if i == 1
  break if i == 2
end
for item, i in ["a", "b"]   # value, index
end
for i in range(0, 3)        # 0, 1, 2
end
5.times { |i| print(i) }
xs.each do |x|
  next if x.nil?            # in a block: skip this element, go on to the next
                            # (2.36.0–2.36.2 under `soli serve`: ended the whole loop — upgrade)
  break if x == :stop       # in a block: stop the iterator
end
xs.map { |x| next if x < 0; x * 2 }   # a skipped element is nil, not dropped
3.times { print("hi") }     # a block may leave out the index
```

No `case/when` (use `match`), no `if … then … else … end` as an expression on
one line, no `loop do`.

## Functions

```soli
def add(a: Int, b: Int) -> Int
  a + b
end

def greet(name)
  "Hi #{name}"
end
greet "Bob"                  # parens optional on calls with args (prefer them)

def kw(a, b: Int = 2)
  a + b
end
kw(1)                        # 3
kw(1, b: 5)                  # 6
invoice.with_fee(fee: 10)    # labels work on method calls too, with the receiver as `this`:
                             # @helper(urgent: true), the bare private _helper(k: v),
                             # Klass.build(k: v), obj&.m(k: v) (a nil receiver gives nil)

def sum_all(numbers: Int[]) -> Int
  numbers.sum
end
sum_all([...xs, 3])          # no *splat params: pass an array
```

Guards: early `return`, then a blank line.

```soli
def status_label(code)
  return "ok" if code == 200
  return "moved" if code == 301

  "error"
end
```

### Lambdas and closures

```soli
double = fn(x) { x * 2 }
halve = |x| { x / 2 }
counter = 0
inc = fn() { counter = counter + 1 }
inc()
inc()                       # counter == 2

def make_adder(n)
  return fn(x) { x + n }    # `return` REQUIRED: a line starting with fn( is
end                         # parsed as a named function declaration
```

An iterator's block may declare fewer parameters than the iterator passes and
gets only those (`xs.each_with_index { |x| … }`, `n.times { … }`,
`h.each { … }`). A one-parameter `reduce` callback gets the `[acc, x]` pair.
A direct call stays strict: `f = fn() { 1 }` then `f(5)` is "Wrong number of
arguments: expected 0, got 1".

A brace-less `fn(x) x * 2` / `|x| x * 2` on its own line swallows the next
statements (parse error at EOF). A line starting with `(` is its own
statement: `(a + b) * 2` is not a call on the previous line. But `def name`
(no parens) whose body starts with `(` → parse error; write `def name()`.

### Blocks

`{ |x| … }` and `do |x| … end` both work on any iterator, including `.map`
assigned to a variable:

```soli
v = [1, 2, 3].map do |x|
  x * 10
end
```

A block's last expression is its value — no `return`.

`break` in a block stops the iterator that called it (`each`, `map`, `times`…),
which answers nil, as in Ruby; in a block passed to your own `def`, that `def`
returns nil.

Passed to your own `def`, a trailing block binds to its `&blk` parameter; with
none, to its last parameter when no argument filled it (`def with_cb(cb)` →
`with_cb() do |x| … end`), else it is ignored (`def k(x = 0, y = 0)` called as
`k(y: 1) do … end` is `[0, 1]`). Same on both engines.

## Classes

```soli
class Person
  name: String
  age: Int
  count: Int = 0            # field default

  new(name: String, age: Int)
    @name = name
    @age = age
  end

  def greet -> String
    "Hello, I'm #{@name}"
  end

  def adult?                # ? and ! suffixes allowed
    @age >= 18
  end

  static def build(name)
    Person.new(name, 20)
  end

  def self.kind             # also a class method
    "person"
  end

  private

  def _secret               # private: callable only on self
    42
  end
end

p1 = Person.new("Ann", 30)
p1.name                     # fields are public
p1.name = "Bo"
Person.build("Z")
Person.kind
```

- Instantiate with `Klass.new(...)`; with no args `Klass.new` and
  `Klass.new()` both build an instance (both engines; `soli check` types it).
- A declared field starts as its default — or `nil` without one — before the
  constructor runs, the parent's first (a subclass default wins); an
  array/hash default is copied per instance. `Klass.new({..})` /
  `Klass.new(k: v)` on a class without a constructor keeps the defaults of the
  fields it is not given. (Up to 2.29.3 the VM raised "Cannot access
  property" on an unassigned declared field.)
- A guard `return` ends a constructor early (`return if amount.nil?`); `new`
  still gives the instance. (Up to 2.29.3 the VM made such a `new` `nil`.)
- `static NAME: Int = 8080` declares a class field; read as `Klass.NAME`
  (`this.NAME` inside static methods). A field given a value needs no type
  (`static LIMIT = 3`, `count = 0`), and an upper-case `NAME = "k"` is a class
  constant like `const NAME = "k"` (since 2.36.0).
- Inherited `Model` instance methods resolve on the implicit self like your
  own: `save` (or `save()`) inside a model method calls `Model#save` — up to
  2.34.0 it raised "Undefined variable 'save'"; write `@save` there.
- `const MAX = 10` in a class body is a class constant (since 2.35.0): a
  bare `MAX` reads it in the class's instance and static
  methods, constructor and the blocks inside them; outside it is `Klass.MAX` or
  `Klass::MAX`, and `@MAX` / `obj.MAX` still work. A parameter or local named
  `MAX` wins in its method. Up to 2.34.0 only `@MAX` worked: bare `MAX` was an
  undefined variable and `Klass.MAX` raised.
- Inheritance: `class B < A`; constructor `super(args)`; parent method
  `super.method_name()`. Calling `super()` inside a regular method does not
  call the parent method.
- `"#{obj}"` prints the inspect form even if you define `to_s`; call
  `obj.to_s` explicitly.
- Calling a `private` method from outside raises
  `private method 'x' called for an instance of Klass`.
- `@@class_vars` are not supported (use a static field).

### Interfaces and abstract classes

```soli
# 2.39.5 and earlier: brace form only, annotations repeated in the class
interface Drawable {
  def draw() -> String;
}

# unreleased on main (after 2.39.5): the `end` form (braces still parse,
# `soli fmt` writes this one), parentheses optional
interface Drawable
  def draw -> String
end

class Circle implements Drawable
  def draw -> String
    "circle"
  end
end
```

- Up to 2.39.5, an implementation must repeat the signature's annotations
  (`def draw` for `-> String` is "does not match interface signature"); after
  2.39.5 an unannotated param/return matches, and a real mismatch names the
  parameter or return that differs.
- A param/field typed with an interface (`repo: Repository`): calling its
  methods (`repo.find(id)`) is a type error up to 2.39.5 ("Cannot access
  member 'find' on type 'Repository'") — leave it unannotated there. Fixed
  after 2.39.5, where a subclass of an implementer is accepted too.
- Runtime: up to 2.39.5 `x.is_a?("Drawable")` is false and an interface name
  is "Undefined variable" — use `x.respond_to?("draw")`. After 2.39.5
  `x.is_a?("Drawable")` is true (class or superclass implements it) and
  `Circle.implements?("Drawable")` exists. `is_a?` always takes a string.
- In an app, after 2.39.5 `soli serve` refuses to boot when a class lacks a
  method of its interface (`soli lint`: `smell/unimplemented-interface`);
  before, nothing checked it in an app.
- After 2.39.5 too: `interface Person extends Named` (a `Person` class is a
  `Named` for params, `is_a?`, `implements?`); `interface Repo<T>` bound by
  `implements Repo<String>` and read through `repo: Repo<String>`.
- After 2.39.5, the checker holds overrides to the parent's signature
  (Liskov): `def area(x: String)` over a parent `area(x: Int)` is an error
  naming `A#area` — unannotated params match, framework parents
  (`Model`, `Controller`) are not judged. And a `def f -> Int` whose last
  line is a String is a static error (it always raised when called).
- Abstract classes (after 2.39.5 only):

```soli
abstract class Shape
  abstract def area -> Float      # no body

  def describe -> String
    "area #{@area}"
  end
end

class Square < Shape
  def area -> Float
    4.0
  end
end
```

  `Shape.new` raises "cannot instantiate abstract class Shape"; a concrete
  subclass missing an `abstract def` is a type error and refuses to boot an
  app; `abstract def` outside an `abstract class` is a parse error.
  `abstract` is not reserved (a variable may be called that). No default
  methods in interfaces — share code via an abstract class or a module.

### Mixins

```soli
module Greetable
  def greet
    "hi #{@name}"
  end
end

class P
  include Greetable
  name: String
  new(name: String)
    @name = name
  end
end
```

Concern hooks (`included do … end`, `class_methods do … end`) exist for models
(see models-orm.md) — unverified here.

### Enums

```soli
enum Color
  Red
  Green
end
c = Color::Red
```

## Pattern matching

```soli
result = match value {
  42 => "the answer",
  "a" => "letter a",
  [first, ...rest] => "#{first} + #{rest.length} more",
  {name: n, age: a} => "#{n} is #{a}",
  n if n > 0 => "positive: #{n}",
  _ => "wildcard"
}
```

Arms are tried in order. A guarded binding (`n if n > 0`) binds anything, so
if it comes before the array/hash arms, an array input reaches `n > 0` and
raises `Cannot compare array and int`. Put structural patterns first.

Under the static checker (`soli script.sl`), matching a value whose type is
known (e.g. `value = [1, 2]` literal) against a hash pattern is a type error
(`expected Hash, found Int[]`); values from parameters/params are untyped and
fine.

## Errors

```soli
try
  risky()
catch e
  print("Error: #{e}")
finally
  cleanup()
end

throw "Division by zero"                   # any value can be thrown
throw {"code": 404, "message": "nope"}     # caught intact: e["code"]
data = fetch() rescue nil                  # postfix rescue
```

There is no `raise` function (`Undefined variable 'raise'`).

A `try` is worth the value of the branch that ran (try body or the catch
clause that took the error; `finally` adds none), since 2.35.0: a method may end in `try … catch … end`, and `port = try … catch e
8080 end` parses. The assigned form runs as a lambda called on the spot — a
`return` inside it leaves only the `try`. Up to 2.34.0 a tail `try` returned
nil and `x = try` was a parse error; use postfix `rescue` there.

`x = if c … elsif … else … end` (and `unless`) is a value the same way
(nil when no branch ran), since 2.36.0; before, only `match`
and the ternary `c ? a : b` were.

An unknown name gets a suggestion when a known one is close (since
2.34.0): `Undefined variable 'countr' (did you mean 'counter'?) at 8:9`,
`Cannot access property 'upcse' on String (did you mean 'upcase'?)`. It also
applies to methods and fields of instances and classes, named arguments,
`soli check` and the `smell/undefined-local` lint rule. The `(did you mean …)`
suffix goes before `at L:C`, so `assert_raises("Undefined variable 'x'")`
fragments still match. Nearer names win a tie (a local before a global), on
both engines (since 2.36.0; before, `--vm` gave a misspelt local nothing).
Errors also carry a stable code (`E0301` undefined variable, `E0304` unknown
member, `E0403` wrong arity, `E0501` division by zero, `E07xx` imports),
shown in the CLI's code frame, not in the message `catch` sees; `soli explain
<code>` describes it.

Typed catch matches class instances (and subclasses), tried in order:

```soli
class InsufficientFunds < DomainError
end

try
  throw InsufficientFunds.new("balance too low", {"needed": 30})
catch DomainError e
  e.message          # "balance too low"
  e.details["needed"] # 30
  e.to_s             # "InsufficientFunds: balance too low"
end
```

Errors raised by Soli itself (index out of bounds, failed `find`, a missing
method) arrive in `catch` as **strings** (`e.class == "string"`), so typed
catches don't match them. Built-in classes: `Error` (with `ValueError`,
`TypeError`, `KeyError`, `IndexError`, `RuntimeError`), `DomainError`.
Uncaught `DomainError` in a request → 422 (verified).
With OpenTelemetry on, any other uncaught error in a request marks the trace's
root span ERROR with an `exception` event (since 2.32.0):
`exception.type` is the thrown instance's class (`PaymentError`), `Error` for
a thrown string or hash, a descriptive name (`TypeError`, `NameError`, …) for
Soli's own errors. Throw an `Error` subclass to get its own group in Datadog
Error Tracking. A `find` miss's 404, `forbidden()`, `halt` and `DomainError`
are not exceptions.

## Modules and imports

```soli
# math.sl
export def add(a, b)
  a + b
end

def _helper            # private to the module, still reachable by add_one
  1
end

export def add_one(a)
  a + _helper()
end

# main.sl
import "./math.sl"
import { add as plus } from "./math.sl"
add(1, 2)
plus(2, 2)
```

All modules share one global namespace: a same-named top-level `def` elsewhere
replaces yours. Inside an app, models/controllers/helpers are auto-loaded —
don't import them.

## Comments

`#` only. `//` still parses but `soli fmt` rewrites it and the docs standard is `#`.
