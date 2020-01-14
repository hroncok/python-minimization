# Python minimization in Fedora

> While Fedora is well suited for traditional physical/virtual workstations and servers, it is often overlooked for use cases beyond traditional installs.
> 
> Some modern types of deployments — such as IoT and containers — are quite sensitive to size. For IoT that's usually slow data connections (for updates/management) and for cloud and containers it’s the massive scale.

-- the preamble of the [Fedora Minimization Objective](https://docs.fedoraproject.org/en-US/minimization/)

One of the biggest things in Fedora is Python. Because [Fedora loves Python](https://fedoralovespython.org/) and because the package manager for Fedora packages -- dnf -- happens to be written in Python, the Python interpreter and its standard library comes pre-installed on many (if not all) Fedora systems and is often not possible to remove it without destroying the system completely or making it unmanageable.

Python comes with [Batteries Included](https://en.wikipedia.org/wiki/Batteries_Included) -- the standard library is quite big. While pleasant for the programmers, this comes with a large filesystem footprint not entirely desired in Fedora. In this document, we will analyze the footprint and offer several minimization solutions/ideas with their challenges, pros (MiB saved) and cons.

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
*77 3.5" floppy disks, courtesy of Dana Walker. Imagine one of them is faulty.*

(All numbers are real installed disk sizes based on the `python38` package installed on Fedora 31, x86_64. The split into subpackages is based on the `python3` package from Fedora 32. Slight differences between Fedora 31 and 32 or between various architectures are irrelevant here, we aim for a long term minimization. See the [source of the numbers][source].)

In Fedora we split the Python interpreter into various RPM subpackages, some of them are optional. This is what you get all the time:

 - `python3` contains `/usr/bin/python3` and friends; has 21 KiB.
 - `python3-libs` contains `/usr/lib64/libpython3.8.so.1.0` and the majority of the standard library, is required by `python3`; has 37.5 MiB.

And this is what you get optionally:

 - `python3-devel` contains the "development files" and makes it possible to compile extension modules, or build RPM packages with Python modules; has 4.5 MiB.
 - `python3-tkinter` contains the `tkinter` module and several others depending on it (e.g. `turtle`), it is *Recommended* (not *Required*) by `python3` when the *tk* framework is installed, to avoid an unnecessary dependency on *tk* and *X*; has 2 MiB.
 - `python3-idle` contains the [Python's Integrated Development and Learning Environment](https://docs.python.org/3/library/idle.html), an application, depends on `tkinter` and is not recommended nor required by anything; has 4.2 MiB.
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

A special case is the `ensurepip` module -- it has only 34.4 KiB, but it *Requires* unbundled `python-pip-wheel` (1.18 MiB) and `python-setuptools-wheel` (348 KiB) - that puts it between (3) and (4) in the above statistics with 1.56 MiB in total.



### File types (and bytecode caches)

The orthogonal dimension is the file type. Python standard library contains directories with both "extension modules" (written in C (usually) and compiled to `*.cpython-38-x86_64-linux-gnu.so` shared object file) and "pure Python" modules (written in Python and saved as `*.py` source file).

Each pure Python module comes in 4 files:

- `module.py` -- the source
- `__pycache__/module.cpython-38.pyc` -- regular (not optimized) bytecode cache
- `__pycache__/module.cpython-38.opt-1.pyc` -- optimized bytecode cache (level 1)
- `__pycache__/module.cpython-38.opt-2.pyc` -- optimized bytecode cache (level 2)

Each of these files has a different purpose (explained below) and each of the files is wasting precious storage space.

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

Python is an interpreted language. As such, when you `import` a pure Python module, it is primarily loaded from the `.py` source. The source is parsed and loaded to Python bytecode, which is stored in memory and executed. To speed things up, the bytecode is cached to special files described below. When the cached bytecode already exists (and considered valid), the module is loaded from there, bypassing the source code.

We currently package the source files and the bytecode cache files as well, but the source files are still needed. They are used in the following ways:

 - module discovery -- the bytecode cache files in `__pycache__` are not importable without the source files;
 - tracebacks -- when Python raises an uncaught exception, it is presented in a form of a *traceback* containing the original source code, loaded from the source files on demand;
 - custom administrator changes and hotfixes -- when editing the source files directly on disk, the bytecode cache is invalidated (at least by default) and will not be used until re-cached;
 - cache invalidation checks -- each time the bytecode is loaded from the cache, the source file is checked for mtime, so it has to exist (there are however [other optional cache invalidation modes](https://docs.python.org/3/reference/import.html#pyc-invalidation) -- checking checksum of the source file or not checking anything);
 - `__file__` -- some modules read the path of their own sources from the magic `__file__` variable and some logic around that might fail if the path is different (such as if the modules is loaded directly from a bytecode cache file).

#### .pyc regular (not optimized) bytecode cache

When a pure Python module gets imported for the first time after it has been modified (or first time ever), the bytecode cache is is created in `__pycache__/<modulename>.cpython-38.pyc` to be later used on subsequent imports. Why are the bytecode cache files created during the buildtime of the RPM `python3` package and shipped with the corresponding `.py` file? This is what would would happen if the files were not shipped:

 1. If a non-root user executes Python code, Python won't succeed saving the file, the bytecode cache will not be written and hence there will be no future benefits from having the cache in the first place -- startup will be slower. On each import, Python will attempt the write which might have further minor negative impact on performance.
 2. If a root user with restricted SELinux context executes Python code, then write operation will fail and the audit log will be pumped with AVC violations. The result is (1) + lots of noise.
 3. If a root user with unrestricted SELinux context runs Python code, Python is able to regenerate and store the `.pyc` files. They will then stay on disk after the package is removed (possibly updated to the next 3.X version) unless proper RPM level trickery is done (such as listing it as `%ghost`).


#### .opt-?.pyc (optimized) bytecode caches

Similarly to the previous point, the optimized bytecode cache files -- `__pycache__/<modulename>.cpython-38.opt-1.pyc` (or `...opt-2.pyc`) -- are created when Python is invoked with the `-O` (or `-OO`) flag.

When run with the optimization flag, [`-O`](https://docs.python.org/3/using/cmdline.html#cmdoption-o):

> Remove assert statements and any code conditional on the value of `__debug__`.

When run with [`-OO`](https://docs.python.org/3/using/cmdline.html#cmdoption-oo):

> Do `-O` and also discard docstrings.

To clarify: This *is* the optimization. There is nothing more. In most common cases, you don't gain any significant performance boost, yet we must assume that there are Fedora users out there invoking Python in this way -- either because their code actually gains performance or because they were tempted by the word "optimization".

The bytecode has asserts, `__debug__` conditionalized code and docstrings (with level 2) optimized away and hence is different and needs a different cache.

If the cache files don't exist and the users invoke Python with `-O`/`-OO` (or other means, such as the [`PYTHONOPTIMIZE`](https://docs.python.org/3/using/cmdline.html#envvar-PYTHONOPTIMIZE) environment variable), everything bad from the previous section would happen.

### Biggest modules in python3-libs, breakdown by file type

module          | .py       | .pyc      | .opt-1.pyc | .opt-2.pyc | other       | total
----------------|-----------|-----------|------------|------------|-------------|---------
encodings       | 1.4 MiB   | 378.4 KiB | 377.9 KiB  | 362.4 KiB  | 24.0 KiB    | 2.5 MiB
pydoc_data      | 656.1 KiB | 408.3 KiB | 408.3 KiB  | 408.3 KiB  | 8.0 KiB     | 1.8 MiB
distutils       | 647.1 KiB | 421.3 KiB | 420.5 KiB  | 321.1 KiB  | 16.9 KiB    | 1.8 MiB
ensurepip       | 7.6 KiB   | 6.5 KiB   | 6.5 KiB    | 5.9 KiB    | 8.0 KiB     | 34.4 KiB<br>+ 1.52 MiB wheels
asyncio         | 441.2 KiB | 365.8 KiB | 363.6 KiB  | 291.2 KiB  | 8.0 KiB     | 1.4 MiB
email           | 364.8 KiB | 283.1 KiB | 282.8 KiB  | 188.7 KiB  | 16.0 KiB    | 1.1 MiB
unicodedata     |           |           |            |            | 1.0 MiB .so | 1.0 MiB
xml             | 288.5 KiB | 242.8 KiB | 241.8 KiB  | 196.4 KiB  | 40.0 KiB    | 1009.5 KiB
lib2to3         | 281.1 KiB | 237.3 KiB | 234.2 KiB  | 185.9 KiB  | 32.0 KiB    | 993.4 KiB
multiprocessing | 262.9 KiB | 222.7 KiB | 220.4 KiB  | 203.1 KiB  | 16.0 KiB    | 925.1 KiB


See the remaining lines in the [data source][source].

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

Our colleague Christian Heimes has proposed [PEP 594](https://www.python.org/dev/peps/pep-0594/) -- *Removing dead batteries from the standard library* for Python upstream. So far, it has not been approved and the discussion [turned out to be a heated one](http://pyfound.blogspot.com/2019/05/amber-brown-batteries-included-but.html). It proposes to remove 30 modules from the standard library for various reasons, mostly because they have better replacements or because they are no longer as useful as they once were.

If approved, this would **save 1.4 MiB / 3.7%** or a bit less (two removed classes are parts of bigger files and the calculations were simplified to assume the entire file is no longer there -- the difference is not significant).

Not to violate the (5) constraint, this however **has to happen in upstream**, that means not sooner than in Python 3.10 (cca Fedora 35). This is not a kind of change that would benefit from pioneering in Fedora.

We are not aware of a static analyzer that would recognize dependencies on standard library modules and there is no existing metadata for this. Just removing the modules in Fedora (or moving them to an optional subpackage) would only cause breakage and break Python users' (1) and Fedora packagers' expectations (3).


### Solution 2: Move developer oriented modules to python3-devel (or split the stdlib into pieces)

Quite a handful of modules are clearly targeted at developers who code in Python and not at the users of the applications written in it.

Here they are, largest first:

 1. `pydoc_data`: Contains data for the `pydoc` module described below.
 2. `distutils`: Used when distributing and installing Python packages trough `setup.py` files. Predecessor of `setuptools`.
 3. `ensurepip`: Used to install `pip`, mostly to virtual environments via the `venv` module.
 4. `lib2to3`: Used by the `2to3` tool to convert legacy code to Python 3. Also used on install time trough `setup.py` files.
 5. `unittest`: A testing framework for unit tests.
 6. `pydoc`: Generates developer documentation from docstrings.
 7. `doctest`: Tests if documentation reflects the reality.
 8. `venv`: Creates Python virtual environments.

Moving all those modules to `python3-devel` (or `python3-libs-devel` etc.) could **save 6.1 MiB / 16%** and additional **1.5 MiB of wheels** (not calculated in the total amount we count percentages from).

This would however **violate the (1) and (3) criterion**. Python users expect working `venv` and `unittest`. Fedora packagers would need to manually (remember: no metadata, no static analyzer) track runtime dependencies on such modules -- they actually happen, for example there are [modules depending on lib2to3](https://pypi.org/project/modernize/).

Alternatively such thing would no longer be allowed to name itself Python. It would merely be a "minimal Python" with a separate entrypoint - and that **violates the (3) or (4) criterion** (depending on the actual implementation).

Alternatively, this change would need to be driven upstream -- track dependencies on standard library modules and allow it to be shipped in parts. See also our draft [PEP 534](https://www.python.org/dev/peps/pep-0534/) -- *Improved Errors for Missing Standard Library Modules*.
If implemented, this wold allow us to split the library to atomic parts and only make the actually used modules mandatory, saving an unknown amount of space (arguably quite large) and several external dependencies as well (such as `libsqlite3.so`, `libgdbm.so` etc.). We could basically do the `python3-tkinter` split at scale, via an upstream supported way.


### Solution 3: Compress large data-like modules

Some pure Python modules, like `encodings` or `pydoc_data` contain mostly data. We could compress the data in the modules.For example `pydoc_data` is basically a dictionary with very long strings. Those strings are repeated in source as well as various bytecode cache files.

We could store them as compressed bytestrings instead.

Alternatively, we could leverage the Python's ability to import from a zip file and zip such modules. That prevents "hot patching" them on live system (constraint (2)), but if absolutely needed, they can be unzipped and edited. The need to live patch `encodings` or `pydoc_data` should not be very common.

Not all modules can be zipped, extra caution would be needed.
For example, `pydoc` currently reads a CSS file like this:

```python
path_here = os.path.dirname(os.path.realpath(__file__))
css_path = os.path.join(path_here, url)
with open(css_path) as fp:
    return ''.join(fp.readlines())
```

Similar code would need to be ported to [importlib.resources](https://docs.python.org/3/library/importlib.html#module-importlib.resources) -- changes like this are very likely accepted by upstream, but still needed to be carefully found first.

Either way, when carefully only zipping `encodings` and `pydoc_data`, we could **save 3.4 MiB / 9 %**. When compressing the strings inline, we anticipate similar or worse result.

### Solution 4: ZIP the entire standard library

Stretching previous solution a bit further, we might want to zip the entire standard library (at least the pure Python parts). However, we are not sure whether this was anticipated by upstream and whether this does not in fact **violate constraint (5)**. Extra care would be needed.

This would require a great deal of testing and thorough analysis of half a million of lines of code.

Not only this will most likely break things, it will probably also **violate constraints (1) and (2)** (Python and Fedora users' expectations). It can also increase the startup time.

To mitigate that, we might want to ship RPM packages with the standard library -- one uncompressed and one zipped:

 - The `python3-libs` package would *Require* any of them (via virtual provides or boolean requires: `Requires: (python3-libs-modules or python3-libs-modules-zip)`).
 - The `python3-lib` package would **Recommend** the uncompressed package.
 - To avoid increasing the total filesystem footprint when both packages are installed, the packages might conflict with each other -- however that might be a bad user experience.

Nevertheless, this might (in theory) **save 17.8 MiB / 47 %**.


### Solution 5: Stop shipping mandatory bytecode cache

This solution sounds simple: We do no longer ship the bytecode cache mandatorily. Technically, we move the `.pyc` files to a subpackage of `python3-libs` (or three different subpackages, that is not important here). And we only *Recommend* them from `python3-libs` -- by default, the users get them, but for space critical Fedora flavors (such as container images) the maintainers can opt-out and so can the powerusers.

This would **save 18.6 MiB / 50%** -- quite a lot.

However, as said earlier, if the bytecode cache files are not there, Python attempts to create them upon first import. That can result in several problems, here we will try to propose how to workaround them.

#### Problem 5.1: Slower starts without bytecode cache

When a non-root user runs Python code, the bytecode cache is never created.

This can result in potentially slower start of Python apps. However, that might be OK: The wast majority of Fedora users will get the *Recommended* bytecode cache and the rest will have a small slowdown. This does not violate users' expectations **if documented properly** - most users get the old behavior (the default remains fast, but big).

Optionally, we might patch Python to warn in that case and suggest installing the appropriate subpackage. That would of course be a downstream only patch and would **violate constraint (5)**. Alternatively, the warning might suggest running a specific command as root to populate the cache -- that might (or might not) be acceptable upstream. Arguably it is not a very nice user experience, and also it only helps with limited bandwith, not limited storage space.

#### Problem 5.2: SELinux denials

When a root user with restricted SELinux context runs Python code, the bytecode cache is not created and the audit log is pumped with AVC violations. The result is the same as in 5.1 plus noise.

As a workaround, we might work with the SELinux experts to allow the Python process to write the bytecode cache even in restricted context.

This could be a **potential security problem** -- any malicious code written in Python would be able to store malicious bytecode in the cache -- all other invocations of Python would execute that bytecode instead of the proper one.

As such, we *think* this **violates constraint (2)** -- Fedora users expect that SELinux keeps them safe. However, we don't really know what level of protection is expected here: This might require further discussions.

#### Problem 5.3: Leftover bytecode cache files

When a root user with unrestricted SELinux context runs Python code, the bytecode cache is created.

As such, it would need to be marked as `%ghost` in the RPM package with the Python source, while it would exist as real file in the RPM package with the bytecode cache.

Example pseudo-specfile snippet:

```spec
%files libs
# this package Recommends the 3 packages below
.../module.py
%dir .../__pycache__/
%ghost .../__pycache__/module.cpython-38.pyc
%ghost .../__pycache__/module.cpython-38.opt-1.pyc
%ghost .../__pycache__/module.cpython-38.opt-2.pyc

%files libs-bytecode-cache
# this package Requires the libs subpackage
.../__pycache__/module.cpython-38.pyc

%files libs-bytecode-cache-opt-1
# this package Requires the libs subpackage
.../__pycache__/module.cpython-38.opt-1.pyc

%files libs-bytecode-cache-opt-2
# this package Requires the libs subpackage
.../__pycache__/module.cpython-38.opt-2.pyc
```

Our experiments show that if two packages co-own a file and one of them is marked as `%ghost`, everything works as expected:

 - manually created `.pyc` file is overridden by the packaged one without a conflict/error/warning/problem
 - manually created `.pyc` file is removed on package removal

Hence, we anticipate this point as potentially non-problematic, however real testing with the `python3` package has not yet been done.

*Note:* If we are to eventually adapt this in all Python RPM packages to gain even more space, this would certainly need more RPM-level abstraction with macros and dark magic (like the debuginfo packages) -- we cannot anticipate all Fedora Python package maintainers to manually do this. However for now, we would only do it in `python3-libs` as written in the goal of this document.


### Solution 6: Stop shipping mandatory optimized bytecode cache

This is essentially the same as previous solution except we would keep the non-optimized bytecode cache mandatory. That gives us several more options to workaround the caveats.

This would **save 11.9 MiB / 32%**.

#### Workaround 6.1: Fallback to less optimized bytecode cache

We can patch Python to fallback to less optimized bytecode cache if the properly optimized bytecode cache does not exist or cannot be created.

 1. opt-2 would fallback to opt-1 or non-optimized (in this order)
 1. opt-1 would fallback to non-optimized
 1. non-optimized would always be present

This workaround would require a change of the current caching logic. Either there will be no attempt to write the new bytecache files if the less optimized bytecode cache exists, or Python would check if it can write the bytecode cache and only fallback to less optimized ones if it cannot write to the destination.

This workaround however **violates Python users' expectations (1)**: It executes less optimized bytecode than the user has elected to. At the same time, this **violates (5)** if done downstream-only. Both can be **solved by doing this with upstream coordination** -- designing a PEP that describes this behavior into great detail, implement the behavior in Fedora and bring it upstream once ready. Impact on performance would need to be evaluated as well.

#### Optimization level 2 is already broken

It is important to note that optimization level 2 bytecode cache in Fedora is already partially "broken". In the times of Python 2 and 3.4 or less, both non-zero optimization levels shared the same bytecode cache paths. Hence the Fedora packages only shipped optimization level 1 `.pyo` files (`o` for optimized).

Python 3.5 has altered the paths to make optimization 1 and 2 cache coexistable and the `python3` package was adapted to ship all 3 levels of optimization (0, 1 and 2), but all the other packages still only ship two (0 and 1) -- [`brp-python-bytecompile` and `%py_byte_compile`](https://docs.fedoraproject.org/en-US/packaging-guidelines/Python_Appendix/#manual-bytecompilation) both only compile for the two levels. That means all the problems with missing bytecode cache files are actually already happening with all Fedora's Python 3 RPM packages (except `python3-libs` itself) when Python is executed with `-OO` or when `PYTHONOPTIMIZE` is set to 2+.

This has been the case **since Fedora 24** and **nobody has ever reported it as a problem** -- hence we might just drop the optimization level 2 bytecode cache and consider the problems an unsupported corner case. That would **save 5.2 MiB / 14%**. Technically this is wrong, but pragmatically it works just fine.

Alternatively, we might make a case upstream and deprecate and remove `-00` because we don't use it -- however we are not sure if that is a good enough reason.

### Solution 7: Stop shipping mandatory source files, ship .pyc instead

Since the `.py` source files are not the ones that are imported by default, we might as well ship only the bytecode files mandatorily.

To allow module discovery, we would need to rename and move the `.pyc` files from `__pycache__/module.cpython-38.pyc` to `../module.pyc`.

When such file is located in `sys.path`, this is what happens:

 - When only `module.py` exists (status quo), everything works as described in the first sections of this document.
 - When both `module.py` and `module.pyc` exist, the `.pyc` is ignored and everything works as if it was not there (including the bytecode cache files in `__pycache__/*.pyc`).
 - When only `module.pyc` exists, the module is imported from that bytecode cache file regardless of the optimization level (bytecode cache files in `__pycache__/*.pyc` are ignored). 

When doing it this way (shipping only nonoptimized `.pyc`, not shipping source or additional bytecode caches (optimized), we would **save 21.7 MiB / 57.9 %**.

Several things would **violate Python/Fedora users' expectations (1)(2)**:

 - Tracebacks would not contain lines of sources.
 - The source files would be gone -- not only users cannot edit them but they can no longer even read them.

To mitigate that, we could have 2 RPM packages with the standard library (similarly to *Solution 4: ZIP the entire standard library*):

 1. One with moved `.pyc` files only.
 2. One with source `.py` files and `__pycache__` (possibly only recommended if combined with other solutions).


In order to save ourselves from 2 conflicting subpackages, we might do it this way:

 1. The moved `.pyc` files package is mandatory.
 2. The other RPM package is recommended.

This however **violates constraint (4)** -- default users would get two files with non-optimized bytecode cache. Unfortunately the files are in different directories, and hence we cannot hardlink them on the RPM level -- RPM only allows hardlinking files in the same directory to avoid cross filesystem hardlinks. If we symlink the files, Python currently does not follow them.

If we get upstream support for following symbolic links, we might do something like this:

```spec
%files libs
# Recommends libs-source
.../module.pyc

%files libs-source
# Requires libs
.../module.py
%dir .../__pycache__/
.../__pycache__/module.cpython-38.pyc  # symbolic link to ../module.pyc
.../__pycache__/module.cpython-38.opt-1.pyc
.../__pycache__/module.cpython-38.opt-2.pyc
```

With the two optimized caches optionally `%ghost`ed if combined with other solutions.

If we don't get upstream support for following symbolic links, we might ship the duplicate bytecode cache files and change them to a hardlink in RPM scriptlet / trigger (if they are on the same filesystem, which is very likely).

Alternatively, we might change the way the source and bytecode caches are prioritized on import time, with upstream coordination, to allow having the non-optimized `.pyc` file in just one location without losing the benefits of having the source files. Such as having an (optionally compressed) source file in a `__pysource__` directory and loading it when showing tracebacks.


### Solution 8: Compress .pyc files

We might propose an upstream change (pioneered in Fedora) to add an option to compress the `.pyc` files. We would add a "compressed" flag to the `.pyc` header, and we would change `importlib` to unzip the payload before unmarshalling (deserializing) the bytecode.

This would potentially save **10.2 MiB / 27.2%**, but it might have negative impact on performance. The number is based on actually zipping each individual `.pyc` file, not on only compressing the content.


### Solution 9: Deduplicate bytecode cache

Given the nature of the bytecode caches, the non-optimized, optimized level 1 and optimized level 2 `.pyc` files may or may not be identical.

Consider the following Python module:

```python
1
```

All three bytecode cache files would by identical.

While with:

```python
assert 1
```

Only the two optimized cache files would be identical with each other.

And this:

```python
"""Dummy module docstring"""
1
```

Would produce two identical bytecode cache files but the opt-2 file would differ.

Only modules like this would produce 3 different files:

```python
"""Dummy module docstring"""
assert 1
```

When we examine all the bytecode cache files currently shipped with `python3-libs` and compare them between the optimization levels, we get:

 - 607 modules have bytecode files
 - 454 identical optimization 0 and 1 pairs
 - 68 identical optimization 1 and 2 pairs
 - 62 identical optimization 0, 1 and 2 triads (already counted in both of the above)

Since all of the bytecode caches are kept within the same folder, we can in fact hardlink them between each other and **save 4.0 MiB / 10.7 %**.

It is also important to realize that most of the standard library modules have docstrings (except empty `__init__.py` files), but only every fourth has `__debug__` conditionals or asserts. If we also go with a solution that removes the second optimization level bytecode cache and combine it with this one, we can deduplicate optimization level 1 bytecode cache for three quarters of the modules.

When the bytecode cache is updated for some reason, e.g. because the source file was updated by an administrator, the cache file is recreated, effectively breaking the hardlink. As more files get updated this way, the size naturally increases, but this does not break users' expectations.

As a nice benefit, we can automatically do this with all Fedora Python RPM packages without any cons (except for an insignificant slowdown when comparing the files during build) saving potentially large amounts of space. Cloud providers will go bankrupt.

As a single data point for that general slim down: On my workstation I have 360 MiB of various Python 3.7 bytecode files in `/usr` and I can save 108 MiB.

### Solution 10: Stop shipping mandatory Python, rewrite dnf to Rust

The main reason we need to ship Python everywhere is the package manager -- dnf. If we rewrite dnf to some non-Python, possibly compiled language such as Rust (or C if we enjoy segfaults), we don't need to ship Python at all. This might sound crazy, but see for example [microdnf](https://github.com/rpm-software-management/microdnf) -- a minimal dnf for (mostly) Docker containers that uses libdnf and hence doesn't require Python.

This solution **saves 37.5 MiB / 100%** of mandatory Python. It possibly also saves more space by reducing the amount of installed Python packages, but increases the size of dnf itself. We can most likely assume a compiled executable would have a lesser footprint than a handful of Python modules used by dnf -- this doesn't violate constraint (4): the combined footprint of (micro)dnf + Python won't be significantly larger than now.

However, most importantly, this solution **violates constraint (2)**: Fedora users expect Python to be available, always. Missing Python could break stuff like Ansible based deployments.


## Conclusion

You can see that some of the solutions offer significant slim-down with very little struggle, while other solutions may turn out to be to breaking. At the same time, various solutions can be combined.

For now, we plan to start with bytecode cache deduplication, and we will let the Fedora community discuss our proposals. After all, there might be holes in them and the list is certainly not complete.


## Copyright

This document is placed in the public domain or under the CC0-1.0-Universal license, whichever is more permissive.

The photos are [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).

[source]: https://github.com/hroncok/python-minimization/blob/master/python-minimization.ipynb
