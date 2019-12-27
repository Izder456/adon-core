
#### Branch  : Master
[![Build Status](https://travis-ci.org/adon-net/adon-core.svg?branch=master)](https://travis-ci.org/adon-net/adon-core)[![Build status](https://ci.appveyor.com/api/projects/status/3n24e2ram3wigsr4/branch/master?svg=true)](https://ci.appveyor.com/project/adon-net/adon-core/branch/master)

#### Branch  : Develop 
[![Build Status](https://travis-ci.org/adon-net/adon-core.svg?branch=develop)](https://travis-ci.org/adon-net/adon-core)[![Build status](https://ci.appveyor.com/api/projects/status/3n24e2ram3wigsr4/branch/develop?svg=true)](https://ci.appveyor.com/project/adon-net/adon-core/branch/develo)


## Building Adon Coin

### On *nix:

Dependencies: GCC 5.5.0 or later, CMake 3+ or later, and Boost 1.55 or later.

You may download them from:

- http://gcc.gnu.org/
- http://www.cmake.org/
- http://www.boost.org/

Alternatively, it may be possible to install them using a package manager.

To build, change to a directory where this file is located, and run `make`. The resulting executables can be found in `build/release/src`.

#### Advanced options:

Parallel build: run `make -j<number of threads>` instead of `make`.

Debug build: run `make build-debug`.

Test suite: run `make test-release` to run tests in addition to building. Running `make test-debug` will do the same to the debug version.

Building with Clang: it may be possible to use Clang instead of GCC, but this may not work everywhere. To build, run `export CC=clang CXX=clang++` before running `make`.

### On Windows:
Dependencies: MSVC 2013 or later, CMake 2.8.6 or later, and Boost 1.55 or later. You may download them from:

- http://www.microsoft.com/
- http://www.cmake.org/
- http://www.boost.org/

To build, change to a directory where this file is located, and run this commands:
```
mkdir build
cd build
cmake -G "Visual Studio 12 Win64" ..
```


And then do Build.

Good luck!
