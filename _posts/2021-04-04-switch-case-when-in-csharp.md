---
layout: post
title:  "Switch Case When In C# Statement And Expression"
tags: C#
---


In this post we are going to take a look at a relatively new feature - `when` keyword in the context of switch statement and switch expression.

**Keyword `when` is used to specify a condition in a `case` label of `switch` statement or expression. This provides a fine-grained control over which switch section will be executed based on a condition (expression which evaluates to boolean).**

Besides the theoretical part, I tried to include many practical and easy to understand examples when this functionality can be useful. Feel free to jump straight there if you just want to look at some examples.


**Contents:**
* TOC
{:toc}


## Overview

Introduction of `when` keyword into the `switch` statement allowed handling more complex scenarios where `case` labels cannot be expressed only with constant or type patterns.

In particular, here is a list of topics we will discuss and also C# versions when they were introduced:

- [Switch statement](#switch-statement-and-when-keyword) - well familiar option to perhaps any programmer, it is present in all C# versions
- [When keyword](#pattern-matching-type-pattern-and-when-keyword) - starting C# 7.0 `when` keyword can be used in switch statement, this post talks a lot about this feature
- [Switch expression](#c-80---using-when-in-switch-expression) - introduced in C# 8.0 and provides `switch`-like semantics in an expression context
- [Relational pattern](#c-90---using-relational-pattern-instead-of-when) - C# 9.0 feature that allows specifying conditions even without `when` keyword

All topics mentioned above are supplemented with a set of examples and common use cases listed in the [Examples of C# Switch Case](#examples-of-c-switch-case) section. This should give a good idea how to use switch-case-when in practice.


## Switch Statement and "when" keyword

Let's take a look at the `switch` statement and how `when` keyword fits in.

**NOTE:** Starting C# 7.0 multiple case labels don't need to be mutually exclusive, and thus the *order of case labels is important*.

### Terminology

The code snippet below uses a very simple switch statement to introduce some terminology we'll use throughout this post. It is quite intuitive but worth mentioning to bring everyone on the same page:

- Match expression
- Case label
- Default label
- Switch section

```csharp
switch (caseSwitch)     // Match Expression - can be any non-null expression
{
    case 1:             // Case Label 1    Switch Section START
    case 2:             // Case Label 2
        // ...
        break;          //                 Switch Section END
    case 3:             // Case Label 3    Switch Section START
        // ...
        break;          //                 Switch Section END
    default:            // Default Label   Switch Section START
        // ...
        break;          //                 Switch Section END
}
```

### Pattern Matching: Type Pattern and "when" keyword

[Type Pattern](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/switch#type-pattern){:target="_blank"} is an interesting feature since it enriches the range of possible use cases where switch case can be applied.

Type Pattern is a new addition to the switch statement pattern matching capabilities in C# 7 which complements an already existing and well-known [constant pattern](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/switch#constant-pattern){:target="_blank"}.

In most cases, it makes sense to use `when` keyword in conjunction with the type pattern. However, it can be use with constant pattern as well but I struggle to come up with meaningful examples.

Below is the syntax illustrated with some pseudo-like code:

```csharp
switch (caseSwitch)
{
    ...
    case TypeA myVar when myVar.Size > 0:
        ...
        break;
    case <type> <variable_name> when <any_boolean_expression>:
        ...
        break;
    ...
}
```


### Fall Through and Variable Scope

It is worth mentioning that similar to regular case labels, labels with `when` keyword also follow the fall through behavior.

Fall through just means that multiple case labels result in one code block execution. Or, in other words, a case section can have multiple case labels.

**NOTE:** A variable defined in a case label is **scoped** to the corresponding **case section**. Please refer to [terminology](#terminology) about case label and case section meaning.

So, the only small caveat here is that variables defined in multiple case labels of the same case section should have different names. Luckily, this is not a problem at all since the compiler helps us by highlighting such errors.

```csharp
switch (caseSwitch)
{
    ...
    case TypeA myVarA when myVarA.Size > 0:
    case TypeB myVarB when myVarB.Color == "red":
        ...
        break;
    ...
}
```


## Examples of C# Switch Case

Now, given we have covered the formal part of the "switch case when" functionality, let's take a look at some common use cases and how `when` keyword comes in handy there.

Examples below are organized in a cheatsheet manner, the main idea is the same in all these examples, just find a match and get a solution!

### Greater Than, Or

Let's start with an extremely simple case, just having a condition that is a comprised of multiple conditions. In the example below we use GreaterThan, Or, LessThan operations.

```csharp
switch (caseSwitch)
{
    case int x when x > 5 || x < 0:    // GreaterThan, Or, LessThan
        // ...
        break;
    case int x when x is > 5 or < 0:   // Newer fancier syntax
        // ...
        break;
    // ...
}
```

### Range or Between

This is accomplished simply by combining multiple constraints in the condition part, quite simple. In the following example we check that integer is in range [0, 100].

```csharp
switch (caseSwitch)
{
    case int x when x >= 0 && x <= 100:    // Standard approach
        // ...
        break;
    case int x when x is >= 0 and <= 100:  // Newer syntax
        // ...
        break;
    // ...
}
```

### Contains

Checking if string contains some substring. The same approach can be applied to collections/arrays.

```csharp
switch (caseSwitch)
{
    case string s when s.Contains("someValue"):
        // ...
        break;
    // ...
}
```

### Null or Empty

Checking whether the string `IsNullOrEmpty`, similarly can be done for `IsNullOrWhiteSpace`.

```csharp
switch (caseSwitch)
{
    case string s when string.IsNullOrEmpty(s):
        // ...
        break;
}
```

### Case Insensitive Comparison

Comparing strings while ignoring case.

```csharp
switch (caseSwitch)
{
    case string s when s.Equals("someValue", StringComparison.InvariantCultureIgnoreCase):
        // ...
        break;
}
```

### StartsWith

Checking if the provided value starts with a particular prefix.

```csharp
switch (caseSwitch)
{
    case string s when s.StartsWith("somePrefix"):
        // ...
        break;
}
```

### Regex

We can even test a string if it matches a regular expression! The example below checks whether it's a string that only consists of alphanumeric characters.

```csharp
switch (caseSwitch)
{
    case string s when Regex.IsMatch(s, @"^[0-9a-zA-Z]+$"):
        // ...
        break;
    // ...
}
```

### Type/typeof

We have used this functionality numerous times throughout this post, just need to combine Type Pattern with `when` keyword.

```csharp
switch (caseSwitch)
{
    case Rectangle x when x.Area == 0:
        // ...
        break;
    // ...
}
```

### Generic Type

As an example, if the provided value is of type `List<int>` and this list is small (less than 10 items), then we want to apply some special handling (e.g. use a brute force algorithm).

```csharp
switch (caseSwitch)
{
    case List<int> list when list.Count < 10:
        // ...
        break;
    // ...
}
```


## C# 8.0 - Using "when" in Switch Expression

Switch Expression is a new construct that provides switch-like semantics in an expression context. In other words, it allows to have similar functionality to a switch statement but it can be used as an expression.

We won't get into details of switch expression but rather see how to use it. To learn more about the switch expression itself please refer to the [docs](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/switch-expression){:target="_blank"}.

The usage is quite similar to a regular switch case, just a few comments:

- `<pattern>` can be Type Pattern or Constant Pattern
- `<case_guard>` is an expression that evaluates to a boolean value
- `<expression>` evaluates to a value

```csharp
var result = caseSwitch switch
{
    <pattern> when <case_guard> => <expression>,    // Expression Arm

    int x when x > 0 => 1,                          // Simple example

    _ **=>** 0                                          // Default Arm
};
```


## C# 9.0 - Using Relational Pattern instead of "when"

Moreover, in some cases we don't even need `when` keyword to specify some of the constrains - C# 9 has a new feature called [Relational Pattern](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/patterns3#relational-patterns){:target="_blank"}.

**IMPORTANT:** Relational Pattern is not a substitute for `when` keyword but rather a nice syntax that in some cases might help write more succinct code.

**NOTES:**

- `<` `<=` `>` `>=` relational operators are supported
- `and` `or` combinators can be used to combine patterns
- Input value should be compared to constant values

For example, let's take a look at how to specify cases if provided value is **in range** or **between** some constant values:

```csharp
var result = price switch
{
    < 10 => "low",
    >= 10 and < 50 => "medium",
    _ => "high"
};
```


## Useful Links

- [Switch statement](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/switch){:target="_blank"}
- [Contextual keyword "when"](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/when){:target="_blank"}
- [Switch expression](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/switch-expression){:target="_blank"}
- [Relational patterns](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-9.0/patterns3#relational-patterns){:target="_blank"}
