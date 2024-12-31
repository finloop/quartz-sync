---
title: Literals and overloading in Python
date: 2023-08-26T21:48:56+02:00
publish: true
tags:
  - python
  - programming
---

It's time to talk about Pythons Literals and I mean that literally :smile:.

![[7b2c231ee55d17dd0dca5bd783823db0_MD5.gif]]

Now that we got that unfunny joke out of the way.

## What are Literals and why are they usefull

The basic motivation behind them is that functions can have arguments that can only take a specific set of values, and those functions return values/types change based on that input. Common examples are (you can find more [here](https://github.com/python/typing/issues/478)):

- `pandas.concat` which can return `pandas.DataFrame` or `pandas.Series`
- `pandas.to_datetime` which can return `datetime.datetime`, `DatetimeIndex`, `Series`, `DataFrame`...

It would be a problem If we couldn't know what type the return value is. Literals can help us indicate that an expression has only a specific value. If we combine that with overloading we can add type hints to those type of functions. But before I'll get to examples that change their return types, let's start with something simple:

```python
from typing import Literal

a: Literal[5] = 5
```

Type checker will know that `a` should always be int 5 and will show a warning if we try to change that:

![[5d5ceceb04ae655118455a8fde7e1b45_MD5.png]]

## More examples

Let's define a function whose return type change depending on the input value. But let's do that without literals and overloading:

```python
def fun(param):
    if param == "all":
        return "all"
    elif param == "number":
        return 1
```

This function takes an argument `param` and returns `all` or number 1. Return type of this function is `Literal["all", 1]`, but if we try to do this:

```python
b = fun("number")
b + 1
```

We get a warning: ![[e84b16fcad958b64865f1eef82a5e831_MD5.png]]

What about this:

```python
b = fun("all")
b  + "all"
```

![[88349dca07a1854b817dca2f7acc6286_MD5.png]]

Type checker doesn't know what is the return type of that function is. We can help him with that by doing an overload.

## Overloading

Overloading in python allows describing functions that have multiple combinations of input and output types (but only **one** definition). You can overload a function using an `overload` decorator like this:

```python
from typing import overload

@overload
def f(a: int) -> int:
   ...
@overload
def f(a: str) -> str:
   ...
def f(a):
   <implementation of f>
```

Create a function first and above it. Then add a series of functions with `@overload` decorators, which will be used to help with guessing return types.

Now back to Literals. How to fix function `fun`? Easy - overload it (and add type hints, just to make sure).

```python
@overload
def fun(param: Literal["all"]) -> Literal["all"]:
    ...
@overload
def fun(param: Literal["number"]) -> int:
    ...
def fun(param: Literal["all", "number"]) -> Literal["all"] | int:
    if param == "all":
        return "all"
    elif param == "number":
        return 1
```

As you can see, this function grew, but we are now able to do this like this:

```python
b = fun("number")
c = b + 1
```

![[66334db98ed6f39044161d5329699aa7_MD5.png]]

without any warnings 😎. And be warned if the return type changes:

```python
b = fun("all")
c = b + 1
```

![[46b42b6b852afe5c234e345ef42d1976_MD5.png]]

## References

- [PEP 586 - Literal Types](https://peps.python.org/pep-0586/)
- [typing#overload](https://docs.python.org/3/library/typing.html#typing.overload)
