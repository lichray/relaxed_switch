<!-- maruku -o switch.html switch.md -->

<style type="text/css">
pre code { display: block; margin-left: 2em; }
ins { text-decoration: none; font-weight: bold; background-color: #A0FFA0 }
del { text-decoration: line-through; background-color: #FFA0A0 }
</style>

<table><tbody>
<tr><th>Doc. no.:</th>	<td>Nnnnn</td></tr>
<tr><th>Date:</th>	<td>2013-01-26</td></tr>
<tr><th>Project:</th>	<td>Programming Language C++, Evolution Working Group</td></tr>
<tr><th>Reply-to:</th>	<td>Zhihao Yuan &lt;lichray at gmail dot com&gt;</td></tr>
</tbody></table>

# Relaxed switch statement

## Motivation

	switch (s) {
	case "add":
		// ...
	}

What is the first time you heard of that the `switch` statement only allows
integer?  The time you want something like above, I suppose?  Well, we have
reasons to place such a limitation before C++11, but now, we have `constexpr`
constructors.  We can trivially relax the expression inside the `switch`
statement to cover more literal types.

## Scope

Some C-like programming languages, like JavaScript, furtherly allows any
expressions to be used in the `case` label.  This proposal does not include
such a change, because the semantics of the `switch` statement has to be
changed from "matching (in any order)" to "comparing from top to bottom" to
ensure the possible side-effects inside the `case` labels.

Although, with the proposed change, the comparison may rely on a user-defined
`operator==` at runtime when matching a `case` label, but the same `operator==`
has to be used at compile-time to ensure that all of the `case` are identical.
So, the solution is to simply ill-form the use of a user-defined literal type
without a `constexpr operator==` in the `switch` statement.

One benefit of having a side-effect-free matching is that the `case` labels
can be matched in any order, hence, as an optimization, the labels can be
sorted at compile-time to enable a binary search at runtime if a `constexpr
operator<` is defined for the literal type.  -- By Gabriel Dos Reis.

## Examples

There are several control logics that can't be achieved in a scalable way with
an "if...else" chain, like fall-through:

	switch (s) {
	case "pre":
		// do something
	case "prefix":
		// do more things
		break;
	}

The code above requires `s` to be of a user-defined literal type.  The perfect
candidate (focusing on C++14) is "string_ref: a non-owning reference to a
string"[\[N3512\]](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2013/n3512.html).

The uniform initialization is allowed in a `case` label.  This idea is
initially driven by "to allow implicit conversion from multiple values".

	switch (e /* std::complex<double>, or std::array<double, 2> */) {
	case { 1.0, 3.0 }:
		break;
	}

If the proposal "User-defined Literals for Standard Library
Types"[\[N3402\]](http://www.open-std.org/JTC1/SC22/WG21/docs/papers/2012/n3402.pdf) gets accepted, the syntax to `switch` on a `std::complex<double>` can be
furtherly simplified:

	switch (e) {
	case 1.0+3.0i:
		break;
	}

I personally suggest the `operator ""s(CharT const *, size_t)` to be assigned
to the literal types, e.g., `string_ref` (or `string_view`, another possible
name), so that we can match arbitrary strings with:

	switch (header) {
	case "MA\0GIC"s:
		// file type determined
	}

The proposal "Relaxed switch statement" itself is fairly simple and does not
depend on the proposals/suggestions motioned above.

## Wording

Change in 4 conv paragraph 2:

> -- When used in the expression of a `switch` statement<ins>, if the given
> type is a class type implicitly convertible to an integral or enumeration
> type</ins>.
> The destination type is integral (6.4).

Change in 6.4 stmt.select paragraph 4:

> The value of a condition that is an initialized declaration in a statement
> other than a `switch` statement
> <ins>with the declared variable of integral or enumeration type, or of that
> variable implicitly converted to integral or enumeration type,</ins>
> is the value of the declared variable contextually converted to `bool` (Clause
> 4). ... The value of a condition that is an expression is the value of the
> expression, contextually converted to `bool` for statements other than
> `switch`
> <ins>with the expression of integral or enumeration type, or of that
> expression implicitly converted to integral or enumeration type</ins>;
> if that conversion is ill-formed, the program is ill-formed. ...

Change in 6.4.2 stmt.switch paragraph 2:

> The condition <del>shall</del> <ins>can</ins> be of integral type, enumeration
> type, or of a class type for which a single non-explicit conversion function
> to integral or enumeration type exists (12.3).  If the condition is of class
> type, the condition is converted by calling that conversion function, and the
> result of the conversion is used in place of the original condition for the
> remainder of this section.  Integral promotions are performed.
> <ins>The promoted type is the destination type.</ins>
> <ins> The condition can also be of floating point type or a literal class
> type which is not implicitly convertible to integral or enumeration type.
> The type of the condition is the destination type.</ins>
> <del>*anything left in this paragraph*</del>

Add two new paragraphs after 6.4.2 stmt.switch paragraph 2:

> Any statement within the `switch` statement can be labeled with one or more
> `case` labels using one of the following syntax:
>> **`case`** _constant-expression_ **`:`**
> where the _constant-expression_ shall be a converted constant expression
> (5.19) of the destination type of the switch condition; and
>> **`case`** _braced-init-list_ **`:`**
> which is equivalent to
>> **`case`** _`t`_ **`:`**
> for some invented temporary variable `t` of the destination type `T`, where
> `constexpr T t = `_`braced-init-list`_.

*Non-editorial Note: There is no conversion rule specified by the _converted
constant expression_ for the floating point type; use the second syntax
`{ 3.0 }` to get a consistent conversion behavior.*

> For a `switch` statement with one or more `case` labels,<br/>
>> `switch (`_s_`) {`<br/>
>> `case `_e<sub>1</sub>_`:` ...<br/>
>> `case `_e<sub>2</sub>_`:` ...<br/>
>> ...<br/>
>> `case `_e<sub>k</sub>_`:` ...<br/>
>> `}`
> the converted constant expressions _e<sub>i</sub>_ `==` _e<sub>i</sub>_,
> `!(`_e<sub>i</sub>_ `<` _e<sub>i</sub>_`)`, and
> _e<sub>i</sub>_ `!=` _e<sub>j</sub>_ where _i &ne; j_, if any, when
> contextually converted to `bool` (4), must yield
> `true` for each _1 &le; i &le; k_ and _1 &le; j &le; k_; otherwise, the
> program is ill-formed.  *\[Note:
> For a literal class type not implicitly convertible to integral or
> enumeration type, at compile-time, a user-defined `constexpr operator==`
> needs to be issued to ensure the uniqueness of
> _e<sub>1</sub> ...  e<sub>k</sub>_, and a user-defined `constexpr operator<`
> can be issued to ensure the sortability of _e<sub>1</sub> ...  e<sub>k</sub>_;
> the same operators shall be selected at runtime. --end note\]*
> The elements `e` of a sorted sequence containing
> _e<sub>1</sub> ...  e<sub>k</sub>_ are partitioned with respect to the
> expression `e < s` and `!(s < e)`, while for all elements `e`, `e < s` shall
> imply `!(s < e)`; otherwise, the behavior is undefined.

*Non-editorial Note: A "switch" statement may be implemented using an
"if...else" chain, a binary search, or even a hash table. The requirements
above ensure that a complexity of log(N) can always be implemented (but not
enforced).*

Add a new grammar in A.5 gram.stmt:

> _labeled-statement:_
>> ...<br/>
>> _attribute-specifier-seq<sub>opt</sub>_ **`case`** _braced-init-list_
>> **`:`** _statement_<br/>
>> ...
