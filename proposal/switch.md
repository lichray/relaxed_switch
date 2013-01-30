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
