# Python minimization in Fedora

> While Fedora is well suited for traditional physical/virtual workstations and servers, it is often overlooked for use cases beyond traditional installs.
> 
> Some modern types of deployments — such as IoT and containers — are quite sensitive to size. For IoT that's usually slow data connections (for updates/management) and for cloud and containers it’s the massive scale.

-- the preamble of the [Fedora Minimization Objective](https://docs.fedoraproject.org/en-US/minimization/)

One of the biggest things in Fedora is Python. Because [Fedora loves Python](https://fedoralovespython.org/) and because the package manager for Fedora packages -- dnf -- happens to be written in Python, the Python interpreter and its standard library comes pre-installed on many (if not all) Fedora systems and is often not possible to remove it without destroying the system completely or making it unmanageable.

Python comes with [Batteries Included](https://en.wikipedia.org/wiki/Batteries_Included) -- the standard library is quite big. While pleasant for the programmers, this comes with a large filesystem footprint not entirely desired in Fedora. In this document, we will analyze the footprint and offer several minimization solutions/ideas with their challenges, pros (MiBs saved) and cons.

**Goal:**

 1. Significantly lower the filesystem footprint of the mandatory Python installation in Fedora.

**Non-goals:**

 1. We don't aim to lower the filesystem footprint of all Python installations in Fedora -- the default may remain big, if there is an opt-out mechanism.
 2. We don't aim to lower the filesystem footprint of all Fedora Python RPM packages, just the `python3` package and its subpackages -- the interpreter and the standard library.

However, if any non-goal becomes a side effect of the solution of our goal, good.

**Constraints:**

 1. Do not break Python users' expectations. As an example, we don't strip Python standard library to the bare minimum and still call it Python.
 2. Do not break Fedora users' expectations. As an example, we don't break the ability to hot patch Python files on a live system by default.
 3. Do not break Fedora packagers' expectations. As an example, we don't [require "system tools" to use a custom Python entrypoint](https://fedoraproject.org/wiki/Changes/System_Python), such as `/usr/libexec/platform-python` or `/usr/libexec/system-python`.
 4. Do not significantly increase the filesystem footprint of the default Python installation. As an example, we don't package [two separate versions (and stacks) of Python](https://fedoraproject.org/wiki/Changes/Platform_Python_Stack) -- one minimal for dnf (or Ansible) and another "normal" for the users.
 5. Do not diverge from upstream significantly (but we can drive upstream change). As an example, we don't reinvent the import machinery of Python downstream only, but we might do it in upstream and even [use Fedora to pioneer the change](https://fedoraproject.org/wiki/Changes/python3_c.utf-8_locale).

## How large is Python, actually

tl;dr Python 3.8.1 in Fedora has 111 MiB (approximately 77 3.5" floppy disks), but we only **install 37.5 MiB by default** (26 floppy disks).

![77 3.5" floppy disks](https://i.imgur.com/exrgnoQ.jpg)
*77 3.5" floppy disks, courtesy of Dana Walker*

(All numbers are real installed disk sizes based on the `python38` package installed on Fedora 31, x86_64. The split into subpackages is based on the `python3` package from Fedora 32. Slight differences between Fedora 31 and 32 or between various architectures are irrelevant here, we aim for a long term minimization. See the [source of the numbers](https://github.com/hroncok/python-minimization/blob/master/python-minimization.ipynb).)

In Fedora we split the Python interpreter into various RPM subpackages, some of them are optional. This is what you get all the time:

 - `python3` contains `/usr/bin/python3` and friends; has 21 KiB.
 - `python3-libs` contains `/usr/lib64/libpython3.8.so.1.0` and the majority of the standard library, is required by `python3`; has 37.5 MiB.

And this is what you get optionally:

 - `python3-devel` contains the "development files" and makes it possible to compile extension modules, or build RPM packages with Python modules; has 4.5 MiB.
 - `python3-tkinter` contains the `tkinter` module and several others depending on it (e.g. `turtle`), it is *Recommended* (not *Required*) by `python3` when the *tk* framework is installed, to avoid an unnecessary dependency on *tk* and *X*; has 2 MiB.
 - `python3-idle` contains the [Python's Integrated Development and Learning Environment](https://docs.python.org/3/library/idle.html), an application, depends on `tkinter` and is not recommended not required by anything; has 4.2 MiB.
 - `python3-test` has the `test` module (the selftest suite of Python) and tests contained in other modules (e.g. `lib2to3.tests`), most users don't need this package, it is the biggest part of Python; has 62.8 MiB.
 - `python-unversioned-command` contains the `/usr/bin/python`  symbolic link; has close to 0 Bytes.

For the sake of this document, we will mostly focus on the `python3-libs` package, as it contains the wast majority of the bytes we want to get rid of from minimal Fedora installations. We will mostly focus on the standard library, not `/usr/lib64/libpython3.8.so.1.0` -- that file has copious 3.7 MiB, but it contains the Python interpreter itself and minimizing that is out of scope here -- we have bigger fish to fry.

## 2-dimensional classification of the standard library files

When we look closely on the files in the standard library, we can classify them by 2 important dimensions: Python modules and file types.

### Python modules

The Python 3.8 standard library has 276 different top-level modules, the biggest two being `test` and `idlelib`, both already not part of `python3-libs`. If we factor out modules and submodules removed from `python3-libs`, the ten larges remaining modules are:

 1. `encodings`: 2.5 MiB
 1. `pydoc_data`: 1.8 MiB
 1. `distutils`: 1.8 MiB
 1. `asyncio`: 1.4 MiB
 1. `email`: 1.1 MiB
 1. `unicodedata`: 1.0 MiB
 1. `xml`: 1010 KiB
 1. `lib2to3`: 993 KiB
 1. `multiprocessing`: 925 KiB
 1. `unittest`: 750 KiB

Some modules here are interesting becasue they contain mostly data (`encodings`, `pydoc_data`, `unicodedata`), or because they are obviously developer oriented and very rarely used on runtime (`distutils`, `lib2to3`, `unittest`).

Special case is the `ensurepip` module - it has only 34.4 KiB, but it *Requires* unbundled `python-pip-wheel` (1.18 MiB) and `python-setuptools-wheel` (348 KiB) - that puts it between (3) and (4) in the above statistics with 1.56 MiB in total.



### File types (and bytecode caches)

The orthogonal dimension is the file type. Python standard library contains directories with both "extension modules" (written in C (usually) and compiled to `*.cpython-38-x86_64-linux-gnu.so` shared object file) and "pure Python" modules (written in Python and saved as `*.py` source file).

Each pure Python module comes in 4 files:

- `module.py` -- the source
- `__pycache__/module.cpython-38.pyc` -- regular (not optimized) bytecode cache
- `__pycache__/module.cpython-38.opt-1.pyc` -- optimized bytecode cache (level 1)
- `__pycache__/module.cpython-38.opt-2.pyc` -- optimized bytecode cache (level 2)

Where each of the file has different purpose (explained below) and each of the files is wasting the precious storage space.

In total, the different file types in `/usr/lib64/python3.8/` take:

 - `.py`: 26.4 MiB
 - `.pyc`: 22.0 MiB
 - `.opt-1.pyc`: 22.0 MiB
 - `.opt-2.pyc`: 19.8 MiB
 - `.so`: 5.3 MiB

Files from `python3-libs` in `/usr/lib64/python3.8/` take:

 - `.py`: 9.8 MiB
 - `.pyc`: 6.7 MiB
 - `.opt-1.pyc`: 6.7 MiB
 - `.opt-2.pyc`: 5.2 MiB
 - `.so`: 4.9 MiB

We see that the various filetypes of pure Python modules occupy significant amount of space when combined. But what are they for?

#### .py source files

Python is an interpreted language. As such, when you `import` a pure Python module, it is primarily loaded from the `.py` source. The source is pasred and loaded to Python bytecode, which is stored in memory and executed. To speed things up, the bytecode is cached to special files described below. When the cached bytecode already exists (and considered valid), the module is loaded from there, bypassing the source code.

We currently package the source files and the bytecode cache files as well, but the source file are still needed. They are used in the following ways:

 - module discovery -- the bytecode cache files in `__pycache__` are not importable without the source files;
 - tracebacks -- when Python raises an uncaught exception, it is presented in a form of a *traceback* containing the original source code, loaded from the source files on demand;
 - custom administrator changes and hotfixes -- when editing the source files directly on disk, the bytecode cache is invalidated (at least by default) and will not be used until re-cached;
 - cache invalidation checks -- each time the bytecode is loaded from the cache, the source file is checked for mtime, so it has to exist (there are however [other optional cache invalidation modes](https://docs.python.org/3/reference/import.html#pyc-invalidation) -- checking checksum of the source file or not checking anything);
 - `__file__` -- some modules read the path of their own sources from the magic `__file__` variable and some logic around that might fail if the path is different (such as if the modules is loaded directly from a bytecode cache file).

#### .pyc regular (not optimized) bytecode cache

When a pure Python module gets imported for the first time after it has been modified (or first time ever), the bytecode cache is is created in `__pycache__/<modulename>.cpython-38.pyc` to be later used on subsequent imports. Why are the bytecode cache files created during the buildtime of the RPM `python3` package and shipped with the corresponding `.py` file? This is what would would happen if the files were not shipped:

 1. If a non-root user executes Python code, Python won't succeed saving the file, the bytecode cache will not be written and hence there will be no future benefits from having the cache in the first place - startup will be slower. On each import, Python will attempt the write which might have further minor negative impact on performance.
 2. If a root user with restricted SELinux context executes Python code, then write operation will fail and the audit log will be pumped with AVC violations. The result is (1) + lots of noise.
 3. If a root user with unrestricted SELinux context runs Python code, Python is able to regenerate and store the `.pyc` files. They will then stay on disk after the package is removed (possibly updated to the next 3.X version) unless proper RPM level trickery is done (such as listing it as `%ghost`).


#### .opt-?.pyc optimized) bytecode caches

Similarly to the previous point, the optimized bytecode cache files -- `__pycache__/<modulename>.cpython-38.opt-1.pyc` (or `...opt-2.pyc`) -- are created when Python is invoked with the `-O` (or `-OO`) flag.

When run with the optimization flag, [`-O`](https://docs.python.org/3/using/cmdline.html#cmdoption-o):

> Remove assert statements and any code conditional on the value of `__debug__`.

When run with [`-OO`](https://docs.python.org/3/using/cmdline.html#cmdoption-oo):

> Do `-O` and also discard docstrings.

To clarify: This *is* the optimization. There is nothing more. In most common cases, you don't gain any significant performance boost, yet we must assume that there are Fedora users out there invoking Python in this way -- either because their code actually gains performance or because they were tempted by the word "optimization".

The bytecode has asserts, `__debug__` conditionalized code and docstrings (with level 2) optimized away and hence is different and needs a different cache.

If the cache files don't exist and the users invoke Python with `-O`/`-OO` (or other means, such as the [`PYTHONOPTIMIZE`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONOPTIMIZE) environment variable), everything bad from the previous section would happen.

### Biggest things

XXX *Present the table from the [Jupyter notebook](https://github.com/hroncok/python-minimization/blob/master/python-minimization.ipynb) here in a fancy way.* XXX

## Possible solutions

Now when we know what is on those 77 floppy disks, we can decide which ones need to go.

![77 3.5" floppy disks](https://i.imgur.com/w9NWTnP.jpg)
*77 3.5" floppy disks, courtesy of Harold Miller. Shall we ditch the beige ones or the blue?*

### Solution 0: Do nothing, keep the status quo

The status quo installs the mandatory 37.5 MiB (out of 111 MiB) by default. This is achieved by splitting various test modules, IDLE and tkinter to separate optional subpackages.

How does that stand? This solution technically discards 51 floppy disks, gets rid of 73.5 MiB, saves 66% of space. That is pretty good. However, it is the status quo and we will use it as base to compare other proposed solutions, hence for the sake of our measurements, this **saves 0 MiB / 0%**. All further percentage savings will be based on the current mandatory 37.5 MiB.

The status quo however already **violates constraint (1)**: it breaks Python users' expectations. As a Python user, I expect the entire of the standard library to be installed, which is not the case. While Python comes pre-installed and ready to be used by developers and users alike, programs using the `tkinter` module will simply fail with a confusing `ModuleNotFoundError`. This has been the case forever and the situation is similar (or worse) with other Linux distributors of Python, such as Debian or openSUSE. Always installing `tkinter` would contradict the goal here, so we won't change that.


### Solution 1: Slim down the Python standard library

One solution is to stop having such a big standard library. Python has existed for some time now and a lot of the standard library modules might no longer be relevant to the general audience.

Our colleague Christian Heimes has proposed [PEP 594](https://www.python.org/dev/peps/pep-0594/) *Removing dead batteries from the standard library* for Python upstream. So far, it has not been approved and the discussion [turned out to be a heated one](http://pyfound.blogspot.com/2019/05/amber-brown-batteries-included-but.html). It proposes to remove 30 modules from the standard library for various reasons, mostly because they have better replacements or because they are no longer as useful as they once were.

If approved, this would **save 1.4 MiB / 3.7%** or a bit less (two removed classes are parts of bigger files and the calculations were simplified to assume the entire file is no longer there - the difference is not significant).

Not to violate the (5) constraint, this however **has to happen in upstream**, that means not sooner than in Python 3.10 (cca Fedora 35). This is not a kind of change that would benefit from pioneering in Fedora.

We are not aware of a static analyzer that would recognize dependencies on standard library modules and there is no existing metadata for this. Just removing the modules in Fedora (or moving them to an optional subpackage) would only cause breakage and break Python users' (1) and Fedora packagers' expectations (3).


### Solution 2: Move large/all developer oriented modules to python3-devel

XXX unittest, lib2to3, ensurepip and venv

XXX breaks Python users' expectations or goes again the separate entrypoint/stack

XXX Static analysis of imports? Both within the stdlib and from RPM packages


### Solution 3: Compress large data-like modules

XXX encodings, pydoc_data

XXX zipimport / compress inline

XXX violates the hot patch constraint, but only for some, can be done upstream

Probably OK, but a lot of manual work.


### Solution 4: ZIP the entire standard library

XXX is that upstream OK?

XXX might break

XXX violates the hot patch constraint - ship two versions of stdlib?

XXX -O ?


### Solution 5: Stop shipping mandatory bytecode cache

XXX make bytecache optional subpackage, recommend if needed

XXX %ghost the files

XXX Fight SELinux?

XXX Patch Python not to attempt the caching?


### Solution 6: Stop shipping mandatory optimized bytecode cache

XXX The same but only for optimized

XXX Patch Python to fallback to nonoptimized bytecode?

XXX Argument: We don't ship opt-2 for other Python packages


### Solution 7: Stop shipping mandatory source files, ship .pyc instead

XXX Needs to be renamed to `<modulename>.pyc`

XXX Tracebacks? By default, we can have them.

XXX Double bytecache?

XXX Conflicting subpackages

XXX Hardlinks created in %post?

XXX Symbolic links with upstream support?

XXX Can be combined with solution 6


### Solution 8: Compress .pyc files

XXX add a "compressed" flag to pyc header, change importlib to unzip payload before unmarshalling

XXX In upstream


### Solution 9: Compress source files

XXX Change traceback (linecache) to unzip sources

XXX violates the live edits constraint


### Solution 10: Deduplicate bytecode cache

XXX Analyze the cache on build time, remove what is "the same"

XXX Most likely doesn't make sense in the stdlib


### Solution 11: Stop shipping mandatory Python, rewrite dnf to Rust

The main reason we need to ship Python everywhere is the package manager -- dnf. If we rewrite dnf to some non-Python, possibly compiled language such as Rust (or C if we enjoy segfaults), we don't need to ship Python at all. This might sound crazy, but see for example [microdnf](https://github.com/rpm-software-management/microdnf) -- a minimal dnf for (mostly) Docker containers that uses libdnf and hence doesn't require Python.

This solution **saves 37.5 MiB / 100%** of mandatory Python. It possibly also saves more space by reducing the amount of installed Python packages, but increases the size of dnf itself. We can most likely assume a compiled executable would have a lesser footprint than a handful of Python modules used by dnf -- this doesn't violate constraint (4): the combined footprint of (micro)dnf + Python won't significantly larger than now.

However, most importantly, this solution **violates constraint (2)**: Fedora users expect Python to be available, always. Missing Python could break stuff like Ansible based deployments.



## Conclusion

While <rust solution> might sound intriguing, it is unfortunately beyond our own ability. And even if we do that, we might want to lower the Python footprint anyway.

Hence, I propose on packaging level, we go explore solution <only bytecode> more deeply and possibly also solution <compress data> -- they don't contradict each other.

In upstream, we will continue to work on <dead batteries>. <compress pyc> would require hard work that might not be needed if we don't ship bytecode files, but if we ship bytecode files only, this might be worth exploring in the future as well. <compress source> sounds like a lot of upstream effort with questionable benefits.
