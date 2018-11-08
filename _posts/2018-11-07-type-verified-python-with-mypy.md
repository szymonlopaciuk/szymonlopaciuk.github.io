---
layout: post
title: "Static Typing in Python 3 with Mypy"
date: 2018-11-07 23:24
---

One of the features of Python 3, as you may be aware of, are type annotations. For example for functions they look like this:

```python
from typing import List

def multisplit(text: str, delimiters: List[str]) -> List[str]:
    for delimiter in delimiters[1:]:
        string = text.replace(delimiter, delimiters[0])

    return string.split(delimiters[0])
```

It might not be very useful in this contrived example, however in some real-life scenarios letting others (or even future yourself) know the API specification can prove really helpful. You might say: if the code is well written, it should be immediately obvious what the types are -- often that is very true, but not always. In addition, with the dawn of tools like PyCharm we can find simple mistakes quicker if the IDE is also aware of the types we use, and parsing docstrings for this is, in my humble opinion, not the most elegant way of doing it. In fact, when working with Python 2, I found myself writing long-winded docstrings often just to describe the accepted and returned types for this exact reason.

Out of the box, type hints are just that: hints. They do not have any more meaning than what we humans understand them to be. Kindly though, the built-in `typing` module supports many composite types (in addition to basic `int`, `str`, etc.) to suit our needs and to create a uniform way of defining more complex structures. There we have things like `List`, `Tuple`, a more general `Iterable`; `AnyStr` for accepting both binary and Unicode strings; `Callable` and more[^1].

So we have established some reasons why having explicitly and clearly defined types is a good idea. But so far, apart from basic IDE functionality, we have no way of enforcing correct typing in our program.

Enter Mypy. Mypy is a static type checker for Python 3: it reads your code and verifies that typing is consistent both internally and against supporting external libraries. You can work Mypy check into your existing testing pipeline to have another extra check for correctness.

So how do we go about actually doing this? Type hinting the code is as easy as my first example. There are a few caveats:

- on variable declaration the type is guessed from the assignment, we might want to extend it sometimes, e.g.:

```python
from typing import Any, List

any_list: List[Any] = [1, 2, 3]  # otherwise will be List[int]
```

- alternatively, we might want to cast a value, e.g.

```python
from typing import cast, Any, List

int_list = [1, 2, 3]
any_list = cast(List[Any], int_list)
```

- when defining a class, we might want to use its type in some type hints on its methods; however, we cannot simply write its name as it is not yet defined:

```python
class MyClass:
    @classmethod
    def get_instance(cls) -> 'MyClass':  # use string instead
        return cls()
```

- ~~an annotation of the return type `-> None` on `__init__` is required~~ [(not anymore)](https://github.com/python/mypy/issues/604#issuecomment-348525995)

- finally, if you are using a virtualenv, you need to tell Mypy where to look for your libraries; below is my script to do that (it additionally skips errors from dependencies and cleans the cache every time it's run):

```bash
export MYPYPATH="${VIRTUAL_ENV}/lib/python3.6/site-packages/"
mypy --follow-imports=silent --no-incremental .
```

[^1]: See more [PEP 484](https://www.python.org/dev/peps/pep-0484/)
