- Feature Name: non_ascii_idents
- Start Date: 2018-06-03
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Allow non-ASCII letters (such as accented characters, Cyrillic, Greek, Kanji, etc.) in Rust identifiers.

# Motivation
[motivation]: #motivation

Rust is written by many people who are not fluent in the English language. Using identifiers in ones native language eases writing and reading code for these developers.

The rationale from [PEP 3131] nicely explains it:

> ~~Python~~ *Rust* code is written by many people in the world who are not familiar with the English language, or even well-acquainted with the Latin writing system. Such developers often desire to define classes and functions with names in their native languages, rather than having to come up with an (often incorrect) English translation of the concept they want to name. By using identifiers in their native language, code clarity and maintainability of the code among speakers of that language improves.
> 
> For some languages, common transliteration systems exist (in particular, for the Latin-based writing systems). For other languages, users have larger difficulties to use Latin to write their native words.

Additionally some math oriented projects may want to use identifiers closely resembling mathematical writing.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Identifiers include variable names, function and trait names and module names. They start with a letter or an underscore and may be followed by more letters, digits and some connecting punctuation.

Examples of valid identifiers are:

* ASCII letters and digits: `image_width`, `line2`, `Photo`, `el_tren`, `_unused`
* words containing accented characters: `garçon`, `hühnervögel`
* identifiers in other scripts: `Москва`, `東京`, ...

Examples of invalid identifiers are:

* Keywords: `impl`, `fn`, `_` (underscore), ...
* Identifiers starting with numbers or containing "non letters": `42_the_answer`, `third√of7`, `◆◆◆`, ...
* Many Emojis: 🙂, 🦀, 💩, ...

Similar Unicode identifiers are normalized: `a1` and `a₁` (a&lt;subscript 1&gt;) refer to the same variable. This also applies to accented characters which can be represented in different ways.

To disallow any Unicode identifiers in a project (for example to ease collaboration or for security reasons) limiting the accepted identifiers to ASCII add this lint to the `lib.rs` or `main.rs` file of your project:

```rust
#![forbid(non_ascii_idents)]
```

Some Unicode character look confusingly similar to each other or even identical like the Latin **A** and the Cyrillic **А**. The compiler may warn you about easy to confuse names in the same scope. If needed (but not recommended) this warning can be silenced with a `#[allow(confusable_non_ascii_idents)]` annotation on the enclosing function or module.

## Usage notes

All code written in the Rust Language Organization (*rustc*, tools, std, common crates) will continue to only use ASCII identifiers and the English language.

For open source crates it is suggested to write them in English and use ASCII-only. An exception can be made if the application domain (e.g. math) benefits from Unicode and the target audience (e.g. for a crate interfacing with Russian passports) is comfortable with the used language and characters. Additionally crates should consider to provide an ASCII-only API.

Private projects can use any script and language the developer(s) desire. It is still a good idea (as with any language feature) not to overdo it.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Identifiers in Rust are based on the [Unicode® Standard Annex #31 Unicode Identifier and Pattern Syntax][UAX31].

Note: The supported Unicode version should be stated in the documentation.

The lexer defines identifiers as:

> **<sup>Lexer:<sup>**  
> IDENTIFIER_OR_KEYWORD:  
> &nbsp;&nbsp; XID_Start&nbsp;XID_Continue<sup>\*</sup>  
> &nbsp;&nbsp; | `_` XID_Continue<sup>+</sup>  
>  
> IDENTIFIER :  
> IDENTIFIER_OR_KEYWORD <sub>*Except a [strict] or [reserved] keyword*</sub>

`XID_Start` and `XID_Continue` are used as defined in the aforementioned standard. The definition of identifiers is forward compatible with each successive release of Unicode as only appropriate new characters are added to the classes but none are removed.

Two identifiers X, Y are considered to be equal if their [NFKC forms][TR15] are equal: NFKC(X) = NFKC(Y).

A `non_ascii_idents` lint is added to the compiler. This lint is `allow` by default. The lint checks if any identifier in the current context contains a codepoint with a value equal to or greater than 0x80 (outside ASCII range). Not only locally defined identifiers are checked but also those imported from other crates and modules into the current context. 

## Confusable detection

Rust compilers should detect confusingly similar Unicode identifiers and warn the user about it.

Note: This is *not* a mandatory for all Rust compilers as it requires considerable implementation effort and is not related to the core function of the compiler. It rather is a tool to detect accidental misspellings and intentional homograph attacks.

A new `confusable_non_ascii_idents` lint is added to the compiler. The default setting is `warn`.

Note: The confusable detection is set to `warn` instead of `deny` to enable forward compatibility. The list of confusable characters will be extended in the future and programs that were once valid would fail to compile.

The confusable detection algorithm is based on [Unicode® Technical Standard #39 Unicode Security Mechanisms Section 4 Confusable Detection][TR39Confusable]. For every distinct identifier X execute the function `skeleton(X)`. If there exist two distinct identifiers X and Y in the same crate where `skeleton(X) = skeleton(Y)` report it.

Note: A fast way to implement this is to compute `skeleton` for each identifier once and place the result in a hashmap as a key. If one tries to insert a key that already exists check if the two identifiers differ from each other. If so report the two confusable identifiers. 

# Drawbacks
[drawbacks]: #drawbacks

* "ASCII is enough for anyone." As source code should be written in English and in English only (source: various people) no charactes outside the ASCII range are needed to express identifiers. Therefore support for Unicode identifiers introduces unnecceray complexity to the compiler.
* "Foreign characters are hard to type." Usually computer keyboards provide access to the US-ASCII printable characters and the local language characters. Characters from other scripts are difficult to type, require entering numeric codes or are not available at all. These characters either need to be copy-pasted or entered with an alternative input method.
* "Foreign characters are hard to read." If one is not familiar with the characters used it can be hard to tell them apart (e.g. φ and ψ) and one may not be able refer to the identifiers in an appropriate way (e.g. "loop" and "trident" instead of phi and psi)
* "My favorite terminal/text editor/web browser" has incomplete Unicode support." Even in 2018 some characters are not widely supported in all places where source code is usually displayed.
* Homoglyph attacks are possible. Without confusable detection identifiers can be distinct for the compiler but visually the same. Even with confusable detection there are still similar looking characters that may be confused by the casual reader.

# Rationale and alternatives
[alternatives]: #alternatives

As stated in [Motivation](#motivation) allowing Unicode identifiers outside the ASCII range improves Rusts accessibility for developers not working in English. Especially in teaching and when the application domain vocabulary is not in English it can be beneficial to use names from the native language. To facilitate this it is necessary to allow a wide range of Unicode character in identifiers. The proposed implementation based on the Unicode TR31 is already used by other programming languages (e.g. Python 3) and is implemented behind the `non_ascii_idents` in *rustc* but lacks the NFKC normalization proposed.

Possible variants:

1. Require all identifiers to be in NFKC or NFC form.
2. Two identifiers are only equal if their codepoints are equal.
3. Perform NFC mapping instead of NFKC mapping for identifiers.
4. Only a number of common scripts could be supported.
5. A [restriction level][TR39Restriction] is specified allowing only a subset of scripts and limit script-mixing within an identifier.

An alternative design would use [Immutable Identifiers][TR31Alternative] as done in [C++]. In this case a list of Unicode codepoints is reserved for syntax (ASCII operators, braces, whitespace) and all other codepoints (including currently unassigned codepoints) are allowed in identifiers. The advantages are that the compiler does not need to know the Unicode character classes XID_Start and XID_Continue for each character and that the set of allowed identifiers never changes. It is disadvantageous that all not explicitly excluded characters at the time of creation can be used in identifiers. This allows developers to create identifiers that can't be recognized as such. It also impedes other uses of Unicode in Rust syntax like custom operators if they were not initially reserved.

It always a possibility to do nothing and limit identifiers to ASCII.

It has been suggested that Unicode identifiers should be opt-in instead of opt-out. The proposal chooses opt-out to benefit the international Rust community. New Rust users should not need to search for the configuration option they may not even know exists. Additionally it simplifies tutorials in other languages as they can omit an annotation in every code snippet.

## Confusable detection

The current design was chosen because the algorithm and list of similar characters are already provided by the Unicode Consortium. A different algorithm and list of characters could be created. I am not aware of any other programming language implementing confusable detection. The confusable detection was primarily included because homoglyph attacks are a huge concern for some member of the community.

Instead of offering confusable detection the lint `forbid(non_ascii_idents)` is sufficient to protect project written in English from homoglyph attacks. Projects using different languages are probably either written by students, by a small group or inside a regional company. These projects are not threatened as much as large open source projects by homoglyph attacks but still benefit from the easier debugging of typos.

# Prior art
[prior-art]: #prior-art

"[Python PEP 3131][PEP 3131]: Supporting Non-ASCII Identifiers" is the Python equivalent to this proposal. The proposed identifier grammar **XID_Start&nbsp;XID_Continue<sup>\*</sup>** is identical to the one used in Python 3.

[JavaScript] supports Unicode identifiers based on the same Default Identifier Syntax but does not apply normalization.

The [CPP reference][C++] describes the allowed Unicode identifiers it is based on the immutable identifier principle.

[Java] also supports Unicode identifiers. Character must belong to a number of Unicode character classes similar to XID_start and XID_continue used in Python. Unlike in Python no normalization is performed.

The [Go language][Go] allows identifiers in the form **Letter (Letter | Number)\*** where **Letter** is a Unicode letter and **Number** is a Unicode decimal number. This is more restricted than the proposed design mainly as is does not allow combining characters needed to write some languages such as Hindi.

# Unresolved questions
[unresolved]: #unresolved-questions

* Which context is adequate for confusable detection: file, current scope, crate?
* Are Unicode characters allowed in `no_mangle` and `extern fn`s?
* How do Unicode names interact with the file system?
* Are crates with Unicode names allowed and can they be published to crates.io?
* Are `non_ascii_idents` and `confusable_non_ascii_idents` good names?
* Should [ZWNJ and ZWJ be allowed in identifiers][TR31Layout]?
* Should *rustc* accept files in a different encoding than *UTF-8*?

[PEP 3131]: https://www.python.org/dev/peps/pep-3131/
[UAX31]: http://www.unicode.org/reports/tr31/
[TR15]: https://www.unicode.org/reports/tr15/
[TR31Alternative]: http://unicode.org/reports/tr31/#Alternative_Identifier_Syntax
[TR31Layout]: https://www.unicode.org/reports/tr31/#Layout_and_Format_Control_Characters
[TR39Confusable]: https://www.unicode.org/reports/tr39/#Confusable_Detection
[TR39Restriction]: https://www.unicode.org/reports/tr39/#Restriction_Level_Detection
[C++]: https://en.cppreference.com/w/cpp/language/identifiers
[Julia Unicode PR]: https://github.com/JuliaLang/julia/pull/19464
[Java]: https://docs.oracle.com/javase/specs/jls/se10/html/jls-3.html#jls-3.8
[JavaScript]: http://www.ecma-international.org/ecma-262/6.0/#sec-names-and-keywords
[Go]: https://golang.org/ref/spec#Identifiers
