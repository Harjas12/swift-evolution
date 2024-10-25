# Raw identifiers

* Proposal: [SE-0451](0451-escaped-identifiers.md)
* Author: [Tony Allevato](https://github.com/allevato); based on [SE-0275](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0275-allow-more-characters-like-whitespaces-and-punctuations-for-escaped-identifiers.md) by [Alfredo Delli Bovi](https://github.com/adellibovi)
* Review Manager: [Joe Groff](https://github.com/jckarter)
* Status: **Active review (October 24...November 7, 2024)**
* Implementation: [swiftlang/swift#76636](https://github.com/swiftlang/swift/pull/76636), [swiftlang/swift-syntax#2857](https://github.com/swiftlang/swift-syntax/pull/2857)
* Review: ([pitch](https://forums.swift.org/t/pitch-revisiting-backtick-delimited-identifiers-that-allow-more-non-identifier-characters/74432))

## Introduction

This proposal adds _raw identifiers_ to the Swift grammar, which are backtick-delimited identifiers that can contain characters other than the current set of identifier-allowed characters in the language.

## Motivation

When naming things in Swift, we use identifiers that are limited to a specific set of characters defined by the [Swift grammar](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/). For the vast majority of identifiers, this set is perfectly reasonable: names of APIs should be no more complex than they need to be, and many natural names do fit within these requirements. However, there do exist some use cases that would be improved by more flexibility in naming:

*   Declarations whose names serve a purely descriptive purpose, like tests.
*   Externally-defined entities that already have natural names that do not fit within Swift's rules for identifiers.

The unifying principle behind these motivating cases is that information is lost or complexity is unnecessarily increased when a developer must forego a more natural name for one that meets Swift's requirements. Indeed, Swift already admits this to a degree by defining the backtick syntax to allow reserved keywords to be used as identifiers.

### Descriptive test naming

When this feature was originally proposed as [SE-0275](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0275-allow-more-characters-like-whitespaces-and-punctuations-for-escaped-identifiers.md), one of the reasons for rejection was that ["[t]he core team feels that this use case could be better handled by test library design that allowed strings to be associated with test cases, rather than try to encode that information entirely in symbol names."](https://forums.swift.org/t/se-0275-allow-more-characters-like-whitespaces-and-punctuations-for-escaped-identifiers/32538/46)

We now have a new testing library in the Swift project called [swift-testing](https://github/com/apple/swift-testing) that implements the feature described above:

```swift
@Test("square returns x * x")
func squareIsXTimesX() {
  #expect(square(4) == 4 * 4)
}
```

Unfortunately, if the user wants to provide a descriptive name for the test, they are now forced to name it **twice**: once in the `@Test` macro and then again in the function itself. As the number of tests increases (see swift-testing's own tests, for [example](https://github.com/swiftlang/swift-testing/blob/main/Tests/TestingTests/ClockTests.swift)), so, too, does the burden on the developer.

This is not merely redundant; it also creates an inconsistency in how the tests are referenced by different tools. Test result reports that are generated by the framework will use the display name, as may user interfaces that display the test hierarchy. However, lower-level tooling such as the compiler itself, the linker, the debugger, index data used for code navigation, and backtraces, will only show the Swift language symbol.

Re-designing the testing framework to eliminate the function name would also not be the correct solution. Consider this hypothetical alternative using a trailing closure that would remove one of the redundant names:

```swift
// Not being proposed
@Test("square returns x * x") {
  #expect(square(4) == 4 * 4)
}
```

This would be unsatisfactory for a few reasons:

*   The current `@Test func` syntax used by swift-testing is a strength because it provides progressive disclosure rather than introducing a new bespoke DSL for tests.
*   There are subtle differences between how the compiler processes closures and regular function declarations that could result in awkward behavior for test authors.
*   The testing framework must still contrive a symbol name for the test by either mangling the user-provided description into something that Swift can support or creating a completely unique name. The user now has no control over that name and no easy way to discover it. That name would still appear in the debugger, index data, and backtraces, so the inconsistency described above still remains.

### Naturally non-alphabetic identifiers

There are some situations where the best name for a particular symbol is numeric or at least starts with a number. As an example, consider a hypothetical color design system that lets users choose a base hue and then a variant/shade of that hue from a fixed set. These design systems often give the color variants numeric names that indicate their intensity—100, 200, and so forth. A naïve API might represent the variant as a numeric value:

```swift
struct Color {
  init(hue: Hue, variant: Int)
}
```

But this API can only enforce correctness at runtime. A developer should naturally reach for an `enum` type for a fixed set of variants like this, but they must transform the name or add unnecessary ceremony to make it fit into Swift's naming rules:

```swift
struct Color {
  init(hue: Hue, variant: ColorVariant)
}

// Not ideal; leading underscore usually means "don't use this"
enum ColorVariant {
  case _50
  case _100
  case _200
  // ...
}
let color = Color(hue: .red, variant: ._100)

// Repetitive
enum ColorVariant {
  case variant50
  case variant100
  case variant200
  // ...
}
let color = Color(hue: .red, variant: .variant100)

// "v" is for... version? No, that's not right.
enum ColorVariant {
  case v50
  // ...
}
```

### Code generators and FFI

Generating code from an external source like a data exchange format specification or transpiling an API from another language is common in programming. When generating such code, names in the original data source may not be usable as Swift identifiers.

Another example is generating type-safe accessors for resources bundled in a library on the filesystem or in an asset catalog. The names of such resources may not necessarily obey Swift conventions—an example is Apple's own _SF Symbols_, which has images with names like `1.circle`.

In these situations, the generator needs to implement a fixed set of transformation rules to convert names that aren't valid identifiers into names that are. One problem with this approach is that such conversions have a cascading effect: the result of converting a Swift-incompatible name into a Swift-compatible one might produce a name that could also be a plausible name in the original source. Therefore, code generators must complicate their logic to avoid such collisions so that an unexpected input doesn't produce invalid code and block the user. This collision-breaking logic often results in arbitrary and difficult-to-understand rules that the user must nevertheless internalize in order to know what to write in the destination language when faced with one of these names in the origin language.

### Module naming at scale

Swift modules (and Clang modules with which Swift interoperates) occupy a single, flat[^1] namespace. The prevailing code organization pattern among many Swift projects is to treat a module as a semantic unit and to give them short, simple names. This approach has already been proven to cause difficulties in real-world builds, as evidenced by the need for the [module aliasing](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0339-module-aliasing-for-disambiguation.md) feature added in Swift 5.7 to reduce the problems that arise when modules with conflicting names occur in the same dependency graph.

[^1]: Clang supports submodules, but they are still _contained_ by their parent modules and importing a parent module also imports all of its (implicit) submodules. This is distinct from having independent compilation units that share a hierarchical namespace. Java packages are an example of the latter.

Experience has shown that this model does not scale well to massively large Swift codebases that use different organizational and build models; for example, companies that use monorepos and build with distributed build systems like Bazel, which we will elaborate on below.

First, many of the projects now using Swift historically used Objective-C or are using common components that are written in Objective-C. That Objective-C code was not written with modules in mind and uses header-file-based inclusions. In order to interoperate with Swift, all of that code would need to be grouped into modules. This would be a significant undertaking to do manually.

Additionally, requiring human developers to choose their own module names has high cognitive overhead and easily leads to conflicts. A component in one part of the codebase can add a new dependency that breaks a different project elsewhere simply by having a conflicting module name with something else the project is already using. Module aliasing has originally designed and implemented does not immediately provide a satisfying remedy here, because it still requires that the project _consuming_ the modules opt into aliasing to resolve the conflict, and it forces them to choose a _second, contrived_ name for the module.

To work around these issues, the approach Bazel has adopted is to automatically derive a module name for a build target based on the unique identifier—a path to its location in the repository—that Bazel already uses to refer to it. This removes the burden from the developer to choose a unique name and reduces the chance of collisions. However, this process still projects a name from a hierarchical namespace onto a flat one, so the build target's natural identifier must be contorted to fit into the identifier-safe naming required by a Swift module. For example, a module defined at the path `myapp/extensions/widget/common/utils` would be given the name `myapp_extensions_widget_common_utils`.

While this works, it is not ideal. Users often know where the code that they want to import lives before they write the line of code that imports it. In order to write the `import` declaration, they must mentally transform the path to the code into the module name, or infrastructure developers must provide tooling to do so. And like the code generation case discussed above, the transformation is not usually reversible, which makes certain kinds of tooling difficult to write and does not  the possibility of collisions. For example, a module named `myapp_extensions_widget_common_utils` could live at `myapp/extensions/widget/common/utils` or at `myapp/extensions/widget/common_utils`. Designing a reversible transform would require making the encoding less friendly to humans who need to read and write those names in their code.

## Proposed Solution

We propose extending the backtick-based syntax used to escape keywords as identifiers to support identifiers that can contain characters other than the standard identifier-safe characters allowed by Swift.

We will distinguish these two uses of backticks with the following terminology:

*   An **escaped identifier** is a sequence of characters surrounded by backticks that starts with `identifier-head` and is followed by zero or more `identifier-character`, as defined in the Swift [grammar](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/summaryofthegrammar/). This is what Swift supports today to treat keywords as identifiers.
*   A **raw identifier** is a sequence of characters surrounded by backticks that contains characters other than those allowed in an escaped identifier. The exact contents are described in [Permissible Characters](#permissible-characters) below.

In both cases, the backticks are **not** considered part of the identifier; they only delimit the identifier from surrounding tokens.

Raw identifiers would provide more flexibility in naming for use cases like the ones discussed earlier in the document as we explain below.

### Describing tests

Test functions could use raw identifiers to describe their purpose clearly without redundancy:

```swift
@Test func `square returns x * x`() {
  #expect(square(4) == 4 * 4)
}
```

Here, the user need only provide a **single** description of the test function. That one name will be used anywhere that the test is referenced: test result reports, the debugger, index data, crash logs, and so forth.

A key observation here is that when using swift-testing, or another framework like XCTest, the name of the test function serves **only to describe the test.** It is _not_ real API, and its only caller is the test framework itself that discovers the function through some dynamic or generative mechanism. Since the name will only be written once at the declaration site, it makes sense to allow for more flexible and verbose naming for the sake of expressibility and simplicity.

Using raw identifiers for test naming fits very well with Swift's philosophy of progressive disclosure. Rather than using a bespoke API for descriptive naming, the author's journey follows a path through the following stages:

* You learn how to write a regular Swift function.
* You learn how to make it a unit test by prepending `@Test` to it.
* You learn that raw identifiers exist and how to apply them to the names of your tests, and this knowledge applies for any identifiers in the language rather than just a specific framework.

### Other kinds of identifiers

Raw identifiers would provide a clean solution to the cases where the most natural name for an identifier is numeric. Considering the color design system again, we could write:

```swift
case ColorVariant {
  case `50`
  case `100`
  case `200`
}
let color = Color(hue: .red, variant: .`100`)
```

One could claim that the backticks are just as much ceremony as using a different prefix, but we believe that raw identifiers are a better choice because they would be using **a common notation** that applies to **all** such symbols. This _reduces_ complexity across all Swift codebases, rather than requiring each developer/API to come up with their own conventions as they do today, which increases complexity for readers.

Raw identifiers could likewise be used for other kinds of foreign identifiers introduced by FFI or code generation. For example, a tool that generates strongly-typed accessors for resources could use raw identifiers to produce a more direct mapping from the resource name to the name in code, reducing complexity by making collisions less likely.

```swift
extension UIImage {
  static var `10.circle`: UIImage { ... }
}
```

Raw identifiers also alleviate the naming problems for modules in very large codebases. Instead of deriving a name using a non-reversible transformation, the build system could simply name the module with the unique identifier that it already associates with that module:

```swift
import `myapp/extensions/widget/common/utils`

// if explicit disambiguation is needed:
`myapp/extensions/widget/common/utils`.SomeClass
```

Doing so greatly improves the experience of developers working in large codebases who can more easily map imports to where the code resides and vice versa, and it also trivializes writing automated tooling that manages build dependencies by scanning imports written in code and updating the build system's definition of the targets.

### Support in other languages

Several other modern programming languages acknowledge the occasional but important need that exists for identifiers that do not meet their standard requirements and support raw identifiers like the ones being proposed here.

In [F#](https://fsharp.org/specs/language-spec/4.1/FSharpSpec-4.1-latest.pdf), identifiers that are surrounded by double backticks and can contain any characters excluding newlines, tabs, and double backticks.

In [Groovy](https://groovy-lang.org/syntax.html#_quoted_identifiers), identifiers after the dot in a dot-expression can be written as quoted string literals containing characters other than those allowed in standard identifiers.

In [Kotlin](https://kotlinlang.org/docs/reference/grammar.html#Identifier), identifiers can be surrounded by single backticks. The characters that are allowed inside backticks may differ based on the target backend (for example, the JVM places additional restrictions). The closest comparison to Swift would be Kotlin/Native, which permits any character other than carriage return, newline, or backtick.

In [Scala](https://www.scala-lang.org/files/archive/spec/2.11/01-lexical-syntax.html), an identifier can contain an arbitrary string when surrounded by single backticks, but host systems may impose some restrictions on which strings are legal for identifiers.

In [Zig](https://ziglang.org/documentation/master/#Identifiers), an identifier that does not meet the standard requirements may be expressed by using an `@` symbol followed by a string literal, such as `@"with a space"`.

## Detailed Design

### Permitted characters

A raw identifier may contain any valid Unicode characters except for the following:

* The backtick (`` ` ``) itself, which termintes the identifier.
* The backslash (`\`), which is reserved for potential future escape sequences.
* Carriage return (`U+000D`) or newline (`U+000A`); identifiers must be written on a single line.
* The NUL character (`U+0000`), which already emits a warning if present in Swift source but would be disallowed completely in a raw identifier.
* All other non-printable ASCII code units that are also forbidden in single-line Swift string literals (`U+0001...U+001F`, `U+007F`).

In addition to these rules, some specific combinations that require special handling are discussed below.

#### Whitespace

A raw identifier may have leading, trailing, or internal whitespace; however, it may not consist of *only* whitespace. "Whitespace" is defined here to mean characters satisfying the Unicode `White_Space` property, exposed in Swift by `Unicode.Scalar.Properties.isWhitespace`.

#### Operator characters

A raw identifier may start with, contain, or end with operator characters, but it may not contain **only** operator characters. To avoid confusion, a raw identifier containing only operator characters is treated as a parsing error: it is neither a valid identifier nor an operator:

```swift
func + (lhs: Int, rhs: Int) -> Int    // ok
func `+` (lhs: Int, rhs: Int) -> Int  // error

let x = 1 + 2    // ok
let x = 1 `+` 2  // error
```

This leaves the door open for a future use of that syntax, should it be desired. There is more discussion of this in [Alternatives Considered](#alternatives-considered).

### Symbol generation and mangling

Backticks are required to delimit a raw identifier written in source code, but they are **not part** of the identifier as it relates to symbols generated by the compiler. For example, ``func `with a space`()`` defines a function whose name is `with a space` without backticks. This is the same as escaped identifiers today: ``func `for`()`` defines a function named `for`, not `` `for` ``.

Fortunately, Swift's symbol mangler already handles non-ASCII-identifier code units in symbol names today by converting them using Punycode. The current algorithm already works for raw identifiers, with one change needed: it does not recognize that an identifier starting with a digit needs to be encoded. Consider the following example:

```swift
// Module "test"
public struct FontWeight {
  public func `100`() { ... }
}
```

Without Punycoding an identifier starting with a digit, the current implementation would mangle the function above as `$s4test10FontWeightV3X00yyF`, which round-trips incorrectly as `test.FontWeight.X00() -> ()`. This proposal would implement the necessary change so that it produces the correct mangling `$s4test10FontWeightV004_100_yyF`.

### Property wrappers

When a property wrapper wraps a variable named with a raw identifier, backticks must also be used to refer to its backing storage or its projected value. The prefix sigil (`_` or `$`, respectively) is part of those identifiers, so it is placed _inside_ the backticks.

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
  var projectedValue: Wrapper<T>
}

struct UsesWrapper {
  @Wrapper var `with a space`: Int
}

let x = UsesWrapper()
print(x.`_with a space`)            // correct
doSomethingWith(x.`$with a space`)  // correct

print(x._`with a space`)            // error
doSomethingWith(x.$`with a space`)  // error
```

The dollar sign (`$`) has two special meanings in Swift when used at the beginning of an identifier:

*   When followed by an integer, it references an unnamed closure argument at that index (e.g., `$0`).
*   When followed by an identifier, it references the projected value of a property wrapper (e.g., `$someBinding`).

We must then decide what an identifier like `` `$0` `` means. The only correct choice is to treat `` `$0` `` as a regular identifier, not a closure argument. This is necessary to allow property wrapper projections to be accessed on properties whose names are numeric raw identifiers:

```swift
@propertyWrapper
struct Wrapper<T> {
  var wrappedValue: T
  var projectedValue: Wrapper<T>
}

let `$1` = "hello"  // error: cannot declare entity named '$1'; the '$' prefix
                    // is reserved for implicitly-synthesized declarations

struct UsesWrapper {
  @Wrapper var `0`: Int

  func f() {
    let closure: (Int) -> Int {
      doSomethingWith(`$0`)  // ok, refers to projected value of `0`
      return $0              // ok, refers to unnamed closure argument
    }
  }
}
```

### Member access expressions

When escaping a keyword as an identifer, Swift allows the backticks to be omitted in certain contexts when the compiler knows that it cannot be a keyword. Most commonly, this occurs with member access expressions:

```swift
enum Access {
  case `public`   // must be escaped here
  case `private`
}

_ = Access.`public`  // ok, but not necessary
_ = Access.public    // also ok
```

Raw identifiers, on the other hand, **must be escaped by backticks in all contexts** since they can contain characters that could be treated as parsing delimiters:

```swift
struct S {
  var `with a space` = 0
}

S().with a space    // error, the parser can't know where the member
                    // name is supposed to end
S().`with a space`  // correct
```

This rule also guarantees an important invariant for tuples: raw identifiers are never confusable with tuple element indices. If a tuple member is accessed with a numeric raw identifier (i.e., `` tuple.`0` ``), that will only interpreted as a tuple element _label_ and never a tuple element _index_. Similarly, if a tuple member is accessed with an unescaped integer (i.e., `tuple.0`), the behavior has not changed: it is only interpreted as an _index_ and never as a _label_, even if there is a label with the same name. For example,

```swift
let x = (0, 1)
_ = x.`0`  // error

let a = (5, `0`: 10)
let b = y.0    // z <- 5
let c = y.`0`  // z <- 10
```

### Objective-C Compatibility

A Swift declaration named with a raw identifier must be given an explicit Objective-C name to export it to Objective-C. The compiler will emit an error if a declaration is given an explicit name with the `@objc(...)` attribute and that name is not a valid Objective-C identifier, or if a declaration is otherwise inferred to be `@objc` and the Swift name of that declaration is not a valid Objective-C identifier.

```swift
@objc class `Class with a Space` {  // error
  @objc(someFunction) func `some function`() {}  // ok
  @objc(`not valid`) func myFunction() {}  // error
}

@objc @objcMembers class AnotherClass {
  var `some property`: Int  // error
}
```

The Objective-C runtime can dynamically support class names and selectors that contain characters that are not expressible in source code, but we do not consider that to be something should be supported. Therefore, such names are forbidden even for symbols that would only be exposed to the runtime but not written to the generated header file.

### Module names

The Clang module map parser already supports modules with non-identifier characters in their names by writing their names in double quotes:

```
module "some/module/name" {
  header "..."
}
```

Such a module would already import cleanly into Swift simply by allowing the module name in an `import` statement to be a raw identifier:

```swift
import `some/module/name`
```

Swift modules pose a different challenge. The compiler's serialization and import search path logic assumes that compiled module artifacts have filenames that match the name of the module; for example, `SomeModule` would be named `SomeModule.swiftmodule`. Using raw identifiers as module names would limit us to only those characters supported in filenames, and these character sets differ between platforms.

There is a feature in the compiler called an "explicit Swift module map" that uses a JSON file to explicitly list all dependencies without using search paths. This can be used to give a module a different name than its filename. However, it does not seem appropriate to restrict the use of raw identifier module names only to users of this feature; a solution should also include users of standard module resolution.

This proposal suggests an alternative: Continue to require that the `-module-name` passed during compilation be a filesystem-compatible name, as is the case today, and then allow aliases set by `-module-alias` to be raw identifiers. This effectively separates the "physical" name of the module (its filesystem name and its ABI name) from what users write in source code. Build systems using this feature would generate the physical names behind the scenes for users and provide the aliases to the compiler, completely transparent to the user. This approach elegantly builds on top of existing features in the compiler and avoids adding more complexity to serialization.

Finally, users who want the ABI name (the name that appears in symbol manglings) to match exactly what users are typing in source code—for example, to ensure consistency between the module names in source and what appears in the debugger and crash logs—could go one step further and pass the raw identifier using the `-module-abi-name` flag.

### Impacts on tooling

During the review of SE-0275, concerns were raised about whether identifiers containing whitespace (or other traditional delimiter characters) would make language tooling harder to write or use. While we cannot address all possible editors here, we believe that modern LSP-driven editors can handle raw identifiers gracefully. One specific concern raised was that double-clicking a word in a raw identifier would not be able to select the whole identifier. Since then, the Language Server Protocol has defined a `textDocument/selectionRange` request that can be used to determine an "interesting" selection range for a text position in a document. If this request is made for the text position that is inside a raw identifier, Swift's language server could return the range of that identifier in the response, allowing the host editor to select it in its entirety.

Likewise, syntax highlighting for raw identifiers is straightforward; they are no more complicated than a single-line string literal.

### Impacts on future reflective APIs

During the review of SE-0275, the point was raised that allowing a name to contain common type delimiters could create ambiguity for future APIs that look up symbols via runtime reflection:

```swift
struct X<T> {
  struct Y {}         // 1
}
struct `X<Int>.Y` {}  // 2

// Does this return 1 or 2?
_ = hypotheticalTypeByName("X<Int>.Y")
```

While these APIs do not yet exist, we feel that they could be implemented unambiguously. Without designing such an API (it is certainly out of scope for this proposal), we can identify some basic principles to address that concern.

First, we can imagine that such an API would—at least at a lower level—allow users to drill down through the module context and its descendant contexts explicitly; for example, something in the style of `myModule.type("X", genericArgs: [swiftStdlibModule.type("Int")]).type("Y")` would be clearly distinct from `myModule.type("X<Int>.Y")`.

Then, if a simplified API like `hypotheticalTypeByName` is desired, we can observe that the type name is being passed in a form such that the library would need the capability to parse a subset of the language's type grammar in order to understand which parts are generic arguments, which are member references, and so forth. Therefore, it stands to reason that the argument to `hypotheticalTypeByName` should be **written as it would be in source**, including any raw identifier delimiters. In doing so, each case is easily distinguished:

```swift
_ = hypotheticalTypeByName("X<Int>.Y")    // returns 1 above
_ = hypotheticalTypeByName("`X<Int>.Y`")  // returns 2 above
```

## Alternatives Considered

[SE-0275](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0275-allow-more-characters-like-whitespaces-and-punctuations-for-escaped-identifiers.md) originally proposed using backticks to support qualified references to operators, such as the following:

```swift
let add: (Int, Int) -> Int = Int.+    // not allowed today
let add: (Int, Int) -> Int = Int.`+`  // would have been allowed by SE-0275
```

As mentioned above, this proposal does not allow this; it is an error to write a raw identifier containing **only** identifier characters. We agree with the Core Team's original feedback that since backticks remove the "magic" around a special character sequence, there is potential for confusion about whether writing `` `+` `` refers to the operator or to a different regular identifier named `+`. If we intended the former, there are also open questions about how one would distinguish prefix, postfix, infix operators with such a syntax. This feature, if desired, demands its own in-depth design and separate proposal.

## Future Directions

There are natural extensions of the raw identifier syntax that could be used to support multi-line identifiers or identifiers containing backticks by adopting a similar syntax to that of raw string literals:

~~~swift
let x = #`contains`some`backticks`#
let y = ##`contains`#single`#pound`#delimiters`##

func ```
  This is a function that multiplies two numbers.
  It's named this way to make sure you REALLY know
  what you're getting into when you multiply two
  numbers.
  ```(_ x: Int, _ y: Int) { x * y }

let fifteen = ```
  This is a function that multiplies two numbers.
  It's named this way to make sure you REALLY know
  what you're getting into when you multiply two
  numbers.
  ```(3, 5)
~~~

At this time, however, we do not believe there are any compelling use cases for such identifiers.

### Escape sequences inside raw identifiers

Raw identifiers follow similar parsing rules as string literals with respect to unprintable characters, which raises the question of how to handle backslashes. The use cases served by many backslash escapes—such as writing unprintable characters—are not desirable for identifiers, so we could choose to treat backslashes as regular literal characters. For example, `` `hello\now` `` would mean the identifier `hello\now`. This could be confusing for users though, who might expect the `\n` to be interpreted the same way that it would be in a string literal. Treating backslashes as literal characters today would also close the door on a viable method of escaping characters inside raw identifiers if we decide that it is needed later. For these reasons, we currently forbid backslashes and leave their purpose to be defined in the future.

## Source compatibility

This proposal is purely additive; it does not affect compatibility with existing source code.

## ABI compatibility

This proposal has no effect on existing ABI. It only makes new valid Swift symbol manglings for symbols that were previously invalid.

## Implications on adoption

For codebases built entirely from source using the same version of the Swift compiler would see no negative impact from using this feature.

For example, if a resilient library used raw identifiers in any declarations that are serialized into the textual interface (e.g., `public` or `@usableFromInline`), that interface would not be consumable by versions of the compiler prior to when this feature was introduced because the older compilers would not be able to parse the identifiers. This, however, is the case for any feature that introduces a new syntax into the language.