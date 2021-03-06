## What is this document?

To clear up some potential confusion: this document is intended as an
overview of the specification of a new tool in the language designer's
toolbox: a hybrid regular expression / grammar that allows users to
match *anything* with as much ease as possible. Its goal is to remain
a proper subset of the Perl 6 Regex specification (embedded in the
deeply Perl6-specific
[STD.pm6](https://github.com/perl6/std/blob/master/STD.pm6)) which is
aimed at being embedded into other languages. You can think of it as the
Perl 6-era version of PCRE.

This document is not a standard or even a very deep specification. It
will gloss over some major integration issues (the biggest that is
called out below is the integration between the target language's
Unicode model and this document), but it should provide a place to
start. Further documents and reference implementations will move
incrementally closer to the end-goal of a universal parsing specification
for all languages!

## Audience

This document is intended as a proposal to
those who develop, extend or maintain languages
_other than Perl 6_ as to how they
could adapt Perl 6 Regexes (AKA Rules) to their language. It
is descriptive of the v1.0.1 DRAFT reference parser.

There is a full JSON parser at the end of this article, written
in this specification's syntax, so if you don't know anything
about what Perl 6 Regexes look like, that might be good
to glance at before continuing.

## History

I measure regular expressions in 5 epochs or generations, with a
zeroth generation for the purely mathematical idea of a regular
expression.

1. In the early days of Unix, there were a few programs that
   used an arcane syntax few had heard of to match text.
   These included ed and grep, soon to be followed by
   ex, vi and friends.
2. Not very long after that, regular expressions were
   "standardized" by extending it and changing/mutilating the
   syntax. This is the POSIX regular expression standard.
3. Then we entered the boom times of regular expression development
   when Perl introduced its improvements to the traditional Unix and
   POSIX regular expressions.
4. Perl, TCL, PCRE, Python and Java all had a hand in the evolution of the
   basic third generation regular expression into what I'll call the
   fourth generation: the regular expressions that you're probably familiar
   with, today. These include features like named groups (first introduced
   in Python) and some implementations have extensions for full Unicode
   support including matching properties, code blocks, etc.
5. And then along came Perl 6. Perl 6 calls its system "regexes" and
   technically they're a full grammar engine, but this can frequently
   be ignored in favor of treating them as souped-up-regexes.

I'll use the term "regex" in this document to mean that 5th (and the 6th that
we're trying to target) generation. This is just convention. Note that
outside of this document, many people use "regex" to refer to legacy regular
expression syntaxes.

## Toward a language-neutral 6th generation

The goal, here, is to come up with the core of Perl 6 Regexes in order
to allow future innovators in other languages to move toward the 6th
generation just as they did when Perl enhanced regular expressions back in
the early 90s.

Nothing here is meant to be locked in to Perl-like syntax. For example,
while braces are used to enclose rules by default, there is nothing
stopping a language like Python from defining rules in terms of
indentation levels. Dollar signs are used for back-referencing, but
this can be thought of as part of the regex syntax, not a generic
variable naming convention, just as backslashed back-references were
treated in previous implementations.

*Note: Since this document was written, the term "Gen6 Regex" has been
introduced. In this document we're refer to both Gen6 Regex and the Perl
6 Regex implementation simply as "regex".*

## Code, not strings

Many languages implement regular expressions as strings that are parsed
at run-time to produce a matching structure. These regexes aren't like that.
They are designed to be first-class code in the native language. As such
they can fail at the time the rest of the program is parsed if malformed
and they can include nested code execution.

Thinking of regexes in this way can be misleading however. There is
an ephemeral nature to this execution that is unlike most other forms of
code execution because of the possibility of backtracking within a
regex, potentially undoing bindings created within match objects and
invalidating previous state.

This is where regexes are difficult for someone familiar with a simple
regular expression library to understand, and it bears some thought,
but one thing that this proposal does to try to limit the impact of that
complexity is to hide everything away behind the `grammar` keyword.
Indeed, though it is a keyword, one could imagine a language implementation
even gating access to it behind a library/module inclusion.

## Keywords and whitespace

Perl 6 is very, very free with introducing keywords. It can do this because
the namespace is pretty well guarded and user-defined values rarely
conflict with these keywords. As such, there are a profusion of ways to
introduce a regex. I'm going to use three basic keywords, here:

* `grammar` - Something like "class" in most modern OO languages, but
  the body isn't a set of properties and methods, but rather regexes.
* `rule` - Introduces a regex that uses significant whitespace by default.
  Occurs within a grammar.
* `token` - Introduces a regex that does not use significant whitespace
  by default. Occurs within a grammar.

### Ratcheting

A substantial difference between both `rule` and `token` and what most
users of older regular expression engines might expect is that they
do not backtrack by default. They merely seek the longest match. If you
want backtracking, you can enable it with `:!ratchet` as described in
the **Adverbs** section, below.

As an example, the rule or token:

    a*ab

Will never match "aab" unless you enable backtracking mode because the
`a*` consumes the "aa" and then the second `a` in the rule/token does
not match. Ratcheting in this way is very useful for parsing, but is
rarely what traditional regular expression users expect, so whatever
mechanism a language uses to integrate regexes, where it appears to be
more traditional and regular-expression-like, it should probably enable
ratcheting. Perl 6 provides an additional keyword,
`regex` which enables backtracking by default (in addition to turning
off significant whitespace).

Note that sub-rules can seem to violate this. A sub-rule represents its own
state, so even though a rule doesn't backtrack, it might reference several
sub-rules that move forward in the input independently. This is a critical
distinction!

### Significant whitespace

What is this significant whitespace thing? An example may help:

Here's a typical regex in just about any modern language:

    \d+ \d+

This is read, in most languages, as "one or more digits followed by a
literal space followed by one or more digits". But inside a rule, it is
actually shorthand for:

    \d+<.ws>\d+

We'll cover the specifics of what that means later, but for now, just
understand that it means "match whitespace smartly".

In tokens, however, literal whitespace is completely ignored unless
quote or escaped. This is the core difference between the two keywords.

Comments are also allowed. I'm not going to go into all of the funky Perl 6
style comment formats, and I will simply include:

    \d+ # Digits

A comment is simply ignored, much like whitespace in a token.

It should also be noted that this behavior is not fixed. The meaning of
significant whitespace doesn't *happen* to coincide with the definition of
`<ws>`, it uses that mechanism, which means that it is user re-definable.
For example, the reference parser redefines `ws` to include comments like
so:

    token ws { <!.ww> [ '#' \N* || \s ]* }

## Literals

Any "word" character which includes alphanumeric characters and underscores
(`_`) can simply be matched as-is:

    token { apple }

But anything more complex must be escaped (prefixed with `\\`) character-by
character or quoted:

    token {
        \. '.' "." # all equivalent
    }

## Simple non-literals

As with most other regular expression formats, `.` matches any
character (including newlines, use `\N` for non-newlines) and there are
the various character class escapes:

* `\c[name]` matches a Unicode codepoint by name
* `\d` matches a digit
* `\h` matches horizontal whitespace
* `\n` matches newlines (actually a "logical newline" which is
  platform-agnostic)
* `\s` matches whitespace
* `\t` matches a tab
* `\v` matches vertical whitespace
* `\w` matches word characters (alphanumeric, underscore)
* `\xnnnn` Matches a Unicode codepoint by 4-digit hex number
  * `\x[n]` Matches a Unicode codepoint by hex number (any length)

All of the above can also be negated by using their upper-case equivalent,
e.g.:

    \w\S*

Matches a word character followed by any non-whitespace.

There are also a more verbose form of these and many other builtin,
named sub-rules:

* `<alpha>` - Alphabetic characters
* `<digit>` - Same as `\d`, a digit
* `<xdigit>` - Hexadecimal digit
* `<alnum>` - Same as `\w`, a word character
* `<punct>` - Punctuation characters
* `<graph>` - `<+alnum+punct>`
* `<space>`- Same as `\s`, a whitespace character
* `<cntrl>` - Control characters
* `<print>` - `<+graph+space-cntrl>`
* `<blank>` - Same as `\h`, horizontal whitespace
* `<lower>` - Lowercase
* `<upper>` - Uppercase
* `<same>` - Between two of the same character
* `<wb>` - A word boundary
* `<ws>` - Smart whitespace (match whitespace and/or a word boundary)
* `<ww>` - Match within a word

When used outside of `<+...>` character classes, these are
sub-rules (see below), and may be preceded (within the `<...>`) by `.` in order to avoid
capturing the matched value, like any other sub-rule. If used as a character
class (e.g. within <+...>) then they can be combined with other character
classes freely except for `same`, `wb`, `ws` and `ww` which are not
character classes.

### Unicode

Unicode properties can also be matched as character classes. For example:

    <:Script('Latin')>

Matches any character with the Unicode "Script" property, "Latin", while:

    <:Block('Basic Latin')>

Matches any character in the "Basic Latin" "Block". These are terms from
the Unicode standard, not from Perl 6.

The following list of properties are supported in both short and long versions
as `<:property>`, but the authoritative source for these properties is
in the
[Unicode Character Database](https://www.unicode.org/reports/tr44/tr44-24.html).

The Unicode Character Database defines properties in terms of a label (such
as "Script") and a value (such as "Latin"). But for those
properties defined as "binary," meaning that they have a value of "Y" or are
simply not defined for the character, the label alone can be used:

* `Cased_Letter` or `LC`
* `Close_Punctuation` or `Pe`
* `Connector_Punctuation` or `Pc`
* `Control or cntrl` or `Cc`
* `Currency_Symbol` or `Sc`
* `Dash_Punctuation` or `Pd`
* `Decimal_Number or digit` or `Nd`
* `Enclosing_Mark` or `Me`
* `Final_Punctuation` or `Pf`
* `Format` or `Cf`
* `Initial_Punctuation` or `Pi`
* `Letter_Number` or `Nl`
* `Letter` or `L`
* `Line_Separator` or `Zl`
* `Lowercase_Letter` or `Ll`
* `Mark` or `M`
* `Math_Symbol` or `Sm`
* `Modifier_Letter` or `Lm`
* `Modifier_Symbol` or `Sk`
* `Nonspacing_Mark` or `Mn`
* `Number` or `N`
* `Open_Punctuation` or `Ps`
* `Other_Letter` or `Lo`
* `Other_Number` or `No`
* `Other_Punctuation` or `Po`
* `Other_Symbol` or `So`
* `Other` or `C`
* `Paragraph_Separator` or `Zp`
* `Private_Use` or `Co`
* `Punctuation or punct` or `P`
* `Separator` or `Z`
* `Space_Separator` or `Zs`
* `Spacing_Mark` or `Mc`
* `Surrogate` or `Cs`
* `Symbol` or `S`
* `Titlecase_Letter` or `Lt`
* `Unassigned` or `Cn`
* `Uppercase_Letter` or `Lu`

*Again, this is a core set of supported labels, but the Unicode standard
is the authoritative source.*

So you might match a title-case string as such:

    token titlestring { <:Titlecase_Letter> <:Lowercase_Letter>* }

To negate a property, simply follow the colon with a `!`.

## Character classes

You can use most of the above character-selecting sub-rules in "character
class" constructions that can only ever match a single character and can
be built up in pieces.

Character classes can be explicit lists of characters:

    <[a..z]> # lower ascii alpha

Or they can be constructed from other classes:

    <+alpha-:Uppercase_Letter>

Which is to say, "the alpha class, but excluding the upper case
letters".

Unicode codepoints can also be referenced by hexadecimal number:

    \x1234 # always four digits

or:

    \x[12345]

Or by name:

    \c[GREEK SMALL LETTER ALPHA]

### A word about ranges

In older regular expression implementations, ranges were specified in
multiple ways such as in character classes, `[a-z]`, or in quantifiers,
`a{2,5}`. Regexes unify these ranges as double-dotted values such as
`<[a..z]>` and `a ** 2..5`.

Perl 6 additionally defines a number of modifiers to ranges that control
whether or not the end-point is included (e.g. `1 ..^ 10` meaning 1 through
9) and a shortcut for numeric ranges that start at zero and do not include
their endpoint (`^n`). These Perl 6 extensions to ranges should be considered
outside of the specification, but implementations may or may not choose to
adopt them or any other Perl 6-specific features.

## Quantifiers

Everything described above is called an "atom". Groups are also atoms.
Any atom can be made optional or repeating in a regex.

The typical regular expression quantifiers are supported:

* `+` - Match the preceding "atom" (single matching literal, group, sub-rule
  or class) one or more times.
* `*` - Match the preceding atom zero or more times.
* `** min[..max]` - Match the preceding atom min to max (\* means forever)
  times.

These constructs modify preceding quantifiers:

* `?` - Match non-greedily
* `!` - Match greedily (the default)
* `:` - Do not perform backtracking for the previous quantifier
* `%` - Match the preceding atom _with quantifier_ separated by the following
  atom (e.g. `<string>+ % ','` which matches comma-separated `<string>` sub-rule
  matches)
* `%%` - Same as `%`, but allows a trailing separator.

Here are some examples:

    a+

Match one or more "a" characters.

    \w*?a

Match zero or more word characters, but match the shortest sequence that
ends in "a".

And finally, there is a special syntax for matching balanced expressions:

    '(' ~ ')' <expression>

Which matches the sub-rule `<expression>` enclosed in parentheses.

This is not substantially different from:

    '(' <expression> ')'

But it does allow the system to provide more useful diagnostics
because it knows that the intent of the user was to match a
balanced beginning and ending token.

## Conjunctions

The `|` of most regular expression syntaxes is now `||`. This will match
on the first matching alternative between the left and right hand sides.

    'a' || 'aa' # will never match the right

While the `|` now matches the longest of the two, as most BNF-like
parser specifications expect, so:

    'a' | 'aa' # matches the right if the input is "aa"

The `&&` conjunction requires that its left and right hand sides both
match the _same substring_. There is a `&` which is mostly identical
to `&&`. It only differs in that `&` and `|` nominally allow evaluation
of their left and right hand sides in any order (and this carries over to
chains of these operators) enhancing potential for parallelization.

## Anchors

* `^` Start of string
* `$` End of string
* `^^` Start of newline-separated line within the string
* `$$` End of newline-separated line within the string
* `<?wb>` Word boundary
* `<!wb>` Not a word boundary
* `<?ww>` Within a word (no boundary)
* `<!ww>` Not within a word
* `<<` Left word boundary (a non-word character or start of string on the
  left and a word character on the right.
* `>>` Right word boundary (a word character on the left and a non-word
  character or end of string on the right)

Note that all of these constructs match an anchor condition, not any of
the text. They are all zero-width.

There are other zero-width assertions that are more complex:

We've already seen some examples of look-around assertions, which are
of the form:

    <?class> # positive look-around

or

    <!class> # negated look-around

In this latter example, do not confuse a look-around, which does not
consume any characters from the input with a negative character class:

    <-class> # negative character class

which does consume the matched character. Look-around matches appear to
match non-zero-width entities, but because they do not consume them,
they are technically zero-width.

You can also turn a full regex into a zero-width assertion in two ways:

    <?before 'apple'> # match before apple, but do not eat it

and

    <?after 'apple'> # match after apple, but do not eat it

In the latter case, if the previous part of the regex had already
consumed the characters matched by the body of the `after`, then
this does not change. The `before` and `after` zero-width assertions
are just that: assertions about the text being matched without any
change in the current position of the match.

Both of these can be negated:

    <!after '.'> <!before '-'>

## Grouping and capturing

A group is specified with parentheses, just like old style regular
expressions:

    'fruit:' ('apple' || 'pear')

Which will match `"fruit:apple"` or `"fruit:pear"`.

Parentheses always capture their matches in the numbered match placeholders
`$n` where n is associated with parenthetical groups in the following way:

* Counting open-parens left-to-right
* But alternation (`|`, `||`, `&` and `&&`) resets the numbering
* If alternations have differing numbers of groups, then the longest
  "wins" in the sense that subsequent groups will be numbered from
  the highest number used thus far.

For example:

    'a' ( ('b') | ('c') ('d') ) ('e')

Would match `"abe"`, capturing `$0` and `$1` as `"b"`, leaving
`$2` undefined and capturing `$3` as "e".

These captured values can be matched later on:

    f (o+) $0

Would match an even number of "o"'s after the "f".

_Note that Perl 6 requires a code assertion between the group and
back-reference in order to cause the back-reference to be available.
This is not part of the generalized specification, and is considered
an implementation detail of Perl 6._

There is also a non-capturing group:

    a [ b || c ]

In this case, grouping is performed, but no capture is saved. This
can improve performance and simplify access to captured groups.

## Named captures

You can also save any match (whether capturing or not) to a specific
name:

    'fruit:' $<fruit> = [ 'apple' || 'pear' ]

In this case, the name, "fruit" is associated with the following
group or other atom as if it were a sub-rule (see below).

## Adverbs

Adverbs are colon-prefixed directives that change the behavior of the
regex engine itself. Their effect lasts until the end of the
inner-most group, rule or token within which they appear. These include:

* `:ignorecase` or `:i` - Match upper and lower-case interchangeably
* `:ignoremark` or `:m` - Match base Unicode characters, ignoring any combining elements (e.g. `:m u` would match "ü")
* `:ratchet` or `:r` - Do not perform _any_ backtracking (default)
* `:sigspace` or `:s` - Treat whitespace as in rules
* `:exhaustive` or `:e` - Perform all possible backtracking, even after a successful match

Any adverb may be negated by using `:!` e.g.:

    'fruit:' [:ignorecase 'apple' | :!ignorecase 'FruitClass']

Would match "fruit:appLE", "fruit:FruitClass" but not "fruit:fruitclass".

Now that you've seen all of the adverbs, we can go over
token vs. rule again. A token is identical to:

    rule foo {:!sigspace :ratchet ...}

While a rule is identical to:

    token foo {:sigspace :!ratchet ...}

## Sub-rules

A sub-rule is like a subroutine call. All `rule` and `token` blocks
introduce a sub-rule, which can be referenced by any other sub-rule
using angle-brackets:

    rule identifier { <word> | <number> }

    token word { \w+ }

    token number { \d+ }

sub-rules capture by default, saving the result under the name of the
sub-rule. If the sub-rule is matched more than once or has a quantifier,
then its capture will be a list of matches. For example, if the above
code also included:

    rule sequence { <identifier>+ % ',' }

Then any comma-separated list of identifiers would be matched with
the named capture under "identifier" containing a list of the
matches that occurred. These captured matches are nested. You can
think of a match as being a data structure that contains a numbered
set of indexes and a named set of indexes, independently. How this
is implemented is language-specific, but a match can contain a whole
tree of matches from sub-rules, and indeed this is how most grammars
work, turning input text into a tree of match results that can be
transformed into an AST or directly acted upon.

To turn off capturing, place a period before the sub-rule name:

    token IPv4 { <.octet> ** 4 % '.' }

So that the resulting match object will not contain the
"octet" entry that would otherwise be created.

If, on the other hand, you want to re-name that entry in the
match, use an equal sign:

    token IPv4 { <byte=octet> ** 4 % '.' }

Recursion is allowed, and is how recursive parsing is performed,
e.g.:

    rule expr {
        <term> | <term> <leftop> <expr>
    }

    rule term {
        <literal> | '(' ~ ')' <expr>
    }

## Code interpolation

There are several ways to interpolate native language structures
and code into a regex:

* `{...}` - Execute the enclosed expression, but ignore its result.
* `<?{...}>` or `<!{...}>` - Execute the enclosed expression and match if it evaluates
  as true (or false if `!` is used).
* `<$variable>` - The contents of `variable` are coerced into a regex
  in a language-specific way and then interpolated into the current regex.
* `<@variable>` - The contents of `variable` are coerced into a list of
  regexes in a language-specific way and then interpolated into the current
  regex as `|` alternations.

The word "expression" is used, here, but what this means should be
re-interpreted in a language-specific way.

All `<...>` interpolations of external code will save to a numbered
group as if they were within parens, but can be named like so:
`<name=?{code}>` or `<name=$variable>`, or their results can
be discarded by prefixing with a period like a sub-rule.

## Integration concerns

While the rest of this document considers what's _in_ a regex, this
section will consider how to take that construct and integrate it into
the larger language. The simplest way to do this is via some form of
method invocation on the grammar as an object. For example, Perl 6
treats the top-level rule `TOP` as special, invoking it via the method
`parse` like so:

    if MyGrammar.parse($string) {
      ...
    }

Ideally such a method should return the match object which contains
the tree of results from the final match.

Another way to do this would be to integrate regexes into an existing
basic regular expression if your language supports them, e.g. (in Perl
5 syntax):

    # start with the "expression" rule
    if (/<MyGrammar::expression>/) {
      ...
    }

But a more interesting approach might be to treat grammars as a
special way of defining classes, and simply allow any sub-rule to
be invoked directly as a method, and perhaps even allow non-regex
methods to be defined on grammars.

How that works and what it means will be very language specific,
but it's there to consider...

### Actions

In Perl 6, grammars are tightly coupled to the concept of "actions".
These are methods of a class associated with the grammar
whose names are the same as the names of the sub-rules in the
grammar. Thus, when a sub-rule match occurs, its corresponding
action can be invoked.

The most common reason to do this is to allow these actions to
augment or replace the match object, for example, performing the
first steps in a code generation pass for a language, creating
a formal abstract syntax tree.

Actions make regexes into a full parser specification system and
so their implementation should be considered carefully in the
implementation of regexes.

### Unicode

Regexes are tightly coupled to Unicode, and there are some hard
choices to be made in integrating the two. Perl 6 obviates many concerns
by transforming all strings into a form called "NFG". How this is
done and what it means is outside of the scope of this document,
but at the very least, the implications of matching a "character"
on a Unicode string that might contain many composing codepoints must
be addressed by the implementation.

### What was dropped

If you look at
[the full Rakudo Perl 6 implementation's Regex
grammar](https://github.com/perl6/nqp/blob/master/src/QRegex/P6Regex/Grammar.nqp),
which is, of course,
written in Perl 6 Regex (using a subset of the language known
as NQP), you can see that many specifics were skipped over.
Some might be added back in later, some just don't make any sense
outside of Perl 6 and some are interesting question marks. I'll
try to outline the high points, here.

* proto/sym - This is a bit like function overloading where
  multiple definitions of the same name are declared with
  different parameters. Here's an example from the core
  NQP definition of regexes themselves:

  ```
  proto token metachar { <...> }
  token metachar:sym<[ ]> { '[' ~ ']' <nibbler> <.SIGOK> }
  token metachar:sym<( )> { '(' ~ ')' <nibbler> <.SIGOK> }
  token metachar:sym<'> { <?['‘‚]> <quote_EXPR: ':q'>  <.SIGOK> }
  token metachar:sym<"> { <?["“„]> <quote_EXPR: ':qq'> <.SIGOK> }
  token metachar:sym<.> { <sym> <.SIGOK> }
  ...
  ```

  Which would then be used as such:

  ```
  rule foo { <metachar> }
  ```

  This is documented under the syntax section of the Perl 6 documentation
  as ["Proto regexes"](https://docs.perl6.org/syntax/Proto%20regexes.html)
  and is probably the most likely feature to be incorporated from this list
  because of the power it lends to sub-classing grammars.
* Perl 6 allows some constructs in places that this spec does not. For
  example, you can save an anchor to a named match variable in Perl 6, but
  in this spec, you can only save (potentially quantified) atoms.
* Inline comments - Perl 6 allows comments to appear
  inline by using bracketing:

  ```
  apple #`(This is a comment) pear
  ```

* [Pod](https://docs.perl6.org/language/pod)
* Angle-bracket quoting of lists of strings as alternation.
  e.g. `< oh happy day >` matches `oh | happy | day`.
* Programmatic range for repetition count
  (e.g. `[foo] ** {min_foo()..max_foo()}`)
* Repetition quantifier variants (e.g. `[foo] ** ^10`, `[foo] ** 0^..^10`, etc)
* Several adverbs:
  * `:Perl5`, `:P5` - (though perhaps in the larger context `:PCRE` might
    have some value?)
  * All of the match-specific adverbs:
    * `:1st`, `:nth` and friends
    * `:continue`, `:continue`
    * `:pos`, `:p`
    * `:exhaustive`, `:ex`
    * `:global`, `:g`
    * `:overlap`, `:o`
  * All of the substitution-specific adverbs:
    * `:samecase` or `:ii`
    * `:samemark` or `:mm`
    * `:samespace` or `:ss`
* The ability to associate adverbs with token/rule declarations
  (this is Perl 6 language syntax, really, not regex syntax).
* Similarly, any additional syntactic elements of the declaration
  of a grammar (e.g. "is export").
* The many other keywords and syntaxes for defining regexes:
  * `regex`
  * `m`
  * `rx`
  * `/.../`
  * All of the bracketing types that can accompany some of these.
* Unicode property union/difference `<:Zp+:Zs-:Zl>`, which is redundant
  with character class union/difference (e.g. `<+:Zp+:Zs+alpha>`).
* Unicode-equivalent operators such as `»` for `>>`
* `<|w>` alias for `<wb>`

### A bit of marketing

There isn't a whole lot of focus in this document on drumming up adoption.
The presumption is that the target audience understands why a full
grammar specification would radically improve the capabilities of any
programming language and the only remaining questions relate to the
difficulty of implementation and disruption to other language elements
(which I _have_ attempted to address). But let's take a moment and
consider the benefits just to make sure everyone is on the same page:

#### Cleanliness

Here is a regular expression that you might find in any modern language:

    <[+-]>?\d*'.'\d+[e<[+-]>?\d+]?

Here is how Perl 6's best practices document suggests you render that
in Perl 6 Regex:

```
token sign { <[+-]> }
token decimal { \d+ }
token exponent { 'e' <sign>? <decimal> }
token float {
    :!ratchet
    <sign>?
    <decimal>?
    '.'
    <decimal>
    <exponent>?
}
```

I think we can all see which is the most readable and maintainable...

#### A full parser

It cannot be stressed enough that Perl 6 Regexes are _not_ just
a new syntax for basic regular expressions. They are full-fledged
parsers (as shown in the JSON example at the end of this document).
Even more so, because of the embedded code, they are fully
context-sensitive parsers, which means that there is no
[formal language](https://en.wikipedia.org/wiki/Chomsky_hierarchy)
that they cannot specify.

This is a quantitative difference in the space of possible problems
that they can solve.

#### Debugging

Because these regexes are first-class language components, they
can be just as debuggable as ordinary code. There is no reason
that a debugger could not step form ordinary code into a regex
and down into enclosed regular code.

They also can produce compile-time errors. In many high level
languages, today, the compile-time/run-time distinction can be
fuzzy, but at the very least, compile-time implies that the
syntax _must_ be correct. For example, consider this Python code:

```Python
import re
def foo(s):
    return re.match(r'fo**', s)
```

This is perfectly valid Python code and unless your test coverage
manages to find the specific case where this bad value is
passed to `re.match` (trivial, here, but obviously that's
because it's a trivial example), you may not find out that there
is an error for a very long time.

In a regex scenario:

```Python
def foo(s):
    grammar Foo:
        token t_foo:
            fo**
    return Foo.t_foo(s)
```

would result in a compile-time error before anyone tried to
execute the code!

### Your regex where?

The first step in porting this spec to your language is
going to be to modify it to be language-specific. To that
end, you will want to start with:

* [the markdown text, available
on GitHub](https://github.com/ajs/Gen6-Regex/blob/master/YOUR-REGEX-HERE-POSTING.md)
* [the formal specification written in itself](https://github.com/ajs/Gen6-Regex/blob/master/regex)
* [the Perl 6 test harness for the formal specification](https://github.com/ajs/Gen6-Regex/blob/master/regex.t6)

## Example

The following is a complete parser for the
[ECMA-404 2nd Edition](http://www.ecma-international.org/publications/files/ECMA-ST/ECMA-404.pdf)
JSON specification:

```
grammar JSON {
    rule TOP {^ <value> $}

    rule value {
        <object> | <array> | <number> | <string> | <name>
    }

    rule name { <true> | <false> | <null> }

    token true  { 'true'  }
    token false { 'false' }
    token null  { 'null'  }

    rule object {
        '{' ( <string> ':' <value> )* % ',' '}'
    }

    rule array {
        '[' <value>* % ',' ']'
    }

    token number {
        '-'?
        [ '0' | <[1..9]> <digit>* ]
        [ '.' <digit>+ ]?
        [:i <[e]> <[\+\-]>? <digit>+ ]?
    }

    token digit { <[0..9]> }

    token string {
        '"' ~ '"' <stringbody>*
    }

    token stringbody {
        # anything non-special:
        <-[\"\\\x[0000]\x[001f]]> |
        # or a valid escape
        '\\' <escape>
    }

    token escape {
        # An escaped special of some sort
        <[\"\\\/bfnr]> |
        # or a Unicode codepoint
        'u' [:i <+[0..9]+[a..f]>] ** 4
    }
}
```

----

*Copyright 2019 by Aaron Sherman*

*This document is distributed under the [Creative Commons
Attribution-ShareAlike 4.0 International
License](http://creativecommons.org/licenses/by-sa/4.0/) and all code
samples herein are alternately licensed under the [Artistic License 2.0](
https://www.perlfoundation.org/artistic-license-20.html).*
