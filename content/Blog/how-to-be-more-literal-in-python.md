---
title: "Literals and overloading in Python"
date: 2023-08-26T21:48:56+02:00
publish: true
---

It's time to talk about Pythons Literals and I mean that literally :smile:.

![Kevin from the office laughing](https://viralviralvideos.com/wp-content/uploads/2017/05/GIF-laughing-funny-LOL-haha-hehe-hilarious-fun-happy-laugh-Kevin-Malone-Brian-Baumgartner-The-Office-GIF.gif)

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

![Pylint warning](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/iw4bv9au4pcfjce4l0qn.png)

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

We get a warning: ![Type warning](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m9a49yb5z51wnefzlh4y.png)

What about this:

```python
b = fun("all")
b  + "all"
```

![Another type warning](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/e2njmdiivptihrsnqxj7.png)

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

![No warnings :)](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o76lqw0asai2313flzu7.png)

without any warnings 😎. And be warned if the return type changes:

```python
b = fun("all")
c = b + 1
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/epb5ck9arndmeokz2deh.png)

## References

- [PEP 586 - Literal Types](https://peps.python.org/pep-0586/)
- [typing#overload](https://docs.python.org/3/library/typing.html#typing.overload)
