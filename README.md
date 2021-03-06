[![Build Status](https://travis-ci.org/jiffyclub/cext23.svg?branch=master)](https://travis-ci.org/jiffyclub/cext23)

# C Extensions for Python 2 and 3

- [Introduction](#introduction)
- [Commonalities](#commonalities)
- [CFFI](#cffi)
- [Cython](#cython)
- [ctypes](#ctypes)
- [Conclusion](#conclusion)
- [Qualifiers](#qualifiers)

## Introduction

Writing pure-Python code that runs unmodified on Python 2 & 3 is doable
with the Python standard library [`__future__` module][future] and
packages like [six][] and [python-future][].
Writing C extensions can be another matter because Python's C-API
[changed substantially][cporting] from Python 2 to 3.
Working with the Python C-API can be a lot of work even when you aren't
trying to support two versions of Python so it's no surprise that options
exist to bypass writing the Python-C interface in straight C.
As a bonus these can also help you communicate with compiled C libraries
using the same code in both Python 2 and 3.
(And [here][snarky] are even more reasons to avoid the Python C-API.)
Below I'll be describing how to wrap C code using
[CFFI][], [Cython][], and [ctypes][].

*Note: I'm writing this as someone who has existing C code that I need to
work with. If you don't already have C and are only looking for performance
improvements then you have [other options][perf-alts] you should explore
before deciding to write C.*

### Help Out!

Pull requests with expanded examples and issues with corrections or
suggestions are appreciated!
I would love for this demo repo to have some more complicated examples,
especially involving NumPy and translating between Python and C data types.
(The integers in the current example map seamlessly between Python
and C, but most other data types do not and it requires some extra setup
in the Python wrapper before sending the data to C.)

## Commonalities

cext23 is a demonstration package with the goal of using different
C interface tools to create identical Python APIs from the same
C source code.
As I describe the interface tools below I'll be linking into the
source code to demonstrate the different aspects of each tool.

In all of the examples discussed the C code being wrapped is exactly
the same and it won't reference Python at all.
All of the Python to C interface layer will be done in Python and/or
generated by one of the tools discussed below.
The C source used in this example repo lives in the [src/](./src/) directory.

Another thing I've done the same way for each example is that even when
the process creates an importable Python extension I've created a
pure-Python wrapper for it.
That allows me to put a nice Python interface + docstring on the function,
maybe use keyword arguments or do some error checking, all in Python
where it's easy.
The goal is to use the Python wrapper to give users a good experience
while handing the core of the computation off to compiled C.

### Testing

Tests can be run using [pytest][] by cloning this repo
and running the command `python setup.py test`.
Running the tests requires the following dependencies:

- CFFI
- Cython
- NumPy

I have only run the tests on Mac and Linux and I don't think the
ctypes example will work on Windows.

## CFFI

From the [CFFI homepage][CFFI]:

> C Foreign Function Interface for Python.
> The goal is to provide a convenient and reliable way to call compiled
> C code from Python using interface declarations written in C.

There are a [number of ways][cffi-overview] to use CFFI,
but when interacting with C code the best strategy is what they call
["API level, out-of-line"][cffi-api-level].
In this mode you write a Python file that describes your extension:
what headers to include, what C code to build with, what libraries to
link with, what functions are part of your interface, etc.
The file is plain Python, but does contain C code in strings.
CFFI uses that Python to create a C file with your Python-C interface defined
and then compiles that into a library file that you can import into Python.
Check out the example CFFI-build file
[cextcffi_build.py](./cext23/cffi/cextcffi_build.py)
and the wrapping Python module [cextcffi.py](./cext23/cffi/cextcffi.py).

Overall using CFFI was a pleasant experience.
I especially like that it handles gathering up all the C code into
a single library/extension via the same interface as the
[distutils Extension class][distutils-ext].
CFFI also has great [setuptools][] [integration][cffi-dist]
to automatically run the extension building as part of installs
(see [setup.py][]).

## Cython

From the [Cython homepage][Cython]:

> Cython is an optimising static compiler for both the Python programming
> language and the extended Cython programming language (based on Pyrex).
> It makes writing C extensions for Python as easy as Python itself.

The Cython process is pretty similar to the CFFI process:
you write a special file that Cython converts to C and that is
then compiled up with other C files into an importable Python library.
In the case of CFFI that special file is written in Python;
with Cython the file is written in [Cython's Python-like langauge][cython-lang].
The cext23 example is in [_cext.pyx](./cext23/cython/_cext.pyx).

Like CFFI, Cython also has [distutils integration][cython-dist] so that
Cython extensions can be automatically built during installation.
When distributing releases of your source code you may note want to
distribute your Cython `.pyx` code, instead you can distribute the
Cython generated `.c` files so that folks can build your project without
having Cython available.
Doing this requires [a little finesse][cython-dist-c].
(Now that Cython is available via things like [Anaconda][] and [Conda][]
it's probably less of a burden on users to have Cython installed.
Binary distribution with [wheels][] and [Conda][] also eliminate the need
for users to compile extensions at all.)

Cython already has broad usage in the scientific Python community where it's
often used to avoid writing plain-C code at all.
(Cython [works very well with NumPy][cython-numpy].)
If you're only looking for a performance gain via C but don't want to
actually write C then Cython is a good choice.
(Though examine [Numba][] as well.)

## ctypes

From the [ctypes][] documentation:

> ctypes is a foreign function library for Python.
> It provides C compatible data types, and allows calling functions in
> DLLs or shared libraries.
> It can be used to wrap these libraries in pure Python.

[ctypes][] is a Python standard library module that allows programmers to
call functions in C libraries from pure-Python modules.
Using ctypes does not involve making a Python C extension,
instead it's calling from pure-Python into plain compiled C.
However, you still need some compiled C to call, and ctypes
does not have any facilities for creating a compiled library
that installs with your Python library.

For this example I used standard [distutils extension features][distutils-ext]
(see [setup.py][]) to create a compiled library that installs in the same
directory as the [cextctypes.py](./cext23/ctypes/cextctypes.py) module.
One challenge of that is that different Python versions on different systems
create files with different names, so it takes some hacks to find the
one to open with ctypes.
I honestly have no idea if this is the correct way to use ctypes,
I expect that ctypes is mostly meant for calling into separately
installed C libraries and not for creating C-Python extensions.

## Conclusion

|     | Strengths | Weaknesses |
| --- | --------- | ---------- |
| [CFFI][] | Builds C extension, plain Python and C | can involve C |
| [Cython][] | Builds C extension, Python-like language | yet another language |
| [ctypes][] | standard library | doesn't help build C extension |

I was impressed with both CFFI and Cython for their ability to compile
my custom C code together with their generated interfaces to create a
complete, importable Python module.
I'm comfortable writing C and I have existing C code so CFFI's
[API discovery magic][cffi-c-magic] is very attractive.
But people who have to write a lot of C interface code
or don't already know C might appreciate the compact,
Pythonic [Cython language][cython-lang].
CFFI and Cython both seem like great options and it's going to take
some more experimentation before I can decide which is best for my
immediate needs, especially when it comes to working with NumPy arrays.

While researching this project I found
[this page][ppug-ext] from the [Python Packaging User Guide][ppug]
very useful.

## Qualifiers

This has been my first experience with CFFI, Cython, and ctypes so
my writeup may contain inaccuracies and/or incompleteness.

[future]: https://docs.python.org/3/library/__future__.html
[six]: https://pythonhosted.org/six/
[python-future]: http://python-future.org/overview.html
[cporting]: https://docs.python.org/3/howto/cporting.html
[snarky]: http://www.snarky.ca/try-to-not-use-the-c-api-directly
[CFFI]: http://cffi.readthedocs.org/
[Cython]: http://cython.org/
[ctypes]: https://docs.python.org/3/library/ctypes.html
[perf-alts]: https://packaging.python.org/en/latest/extensions/#alternatives-to-handcoded-accelerator-modules
[pytest]: https://pytest.org/
[cffi-overview]: https://cffi.readthedocs.org/en/latest/overview.html
[cffi-api-level]: https://cffi.readthedocs.org/en/latest/overview.html#real-example-api-level-out-of-line
[cffi-c-magic]: https://cffi.readthedocs.org/en/latest/cdef.html#letting-the-c-compiler-fill-the-gaps
[distutils-ext]: https://docs.python.org/3/distutils/apiref.html#distutils.core.Extension
[setuptools]: https://pythonhosted.org/setuptools/index.html
[cffi-dist]: https://cffi.readthedocs.org/en/latest/cdef.html
[setup.py]: ./setup.py
[cython-lang]: http://docs.cython.org/src/userguide/language_basics.html
[cython-dist]: http://docs.cython.org/src/reference/compilation.html#compiling-with-distutils
[cython-dist-c]: http://docs.cython.org/src/reference/compilation.html#distributing-cython-modules
[cython-numpy]: http://docs.cython.org/src/tutorial/numpy.html
[Anaconda]: https://store.continuum.io/cshop/anaconda/
[Conda]: http://conda.pydata.org/docs/
[wheels]: https://wheel.readthedocs.org/en/latest/
[Numba]: http://numba.pydata.org/
[ppug-ext]: https://packaging.python.org/en/latest/extensions/
[ppug]: https://packaging.python.org/en/latest/
