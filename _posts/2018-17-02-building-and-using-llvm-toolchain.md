---
published: true
---
## Building and using llvm toolchain

This post can be regarded as a supplement to Chandler Carruth's talk on [LLVM: A Modern, Open C++ Toolchain](https://www.youtube.com/watch?v=uZI_Qla4pNA).

Anybody willing to start playing with llvm toolchain shall better start with this talk. A must watch.
In the talk, Chandler demonstrates building llvm toolchain and using a number of programming aids offered by the llvm toolchain.

Here is the list of commands extracted from the talk to build llvm toolchain.

```bash
# set LLVM_ROOT to the folder where llvm files are going to stay.
cd $LLVM_ROOT
git clone --depth=1 https://llvm.org/git/llvm.git
cd llvm/tools
git clone --depth=1 https://llvm.org/git/clang.git
git clone --depth=1 https://llvm.org/git/lld.git
cd clang/tools
git clone --depth=1 https://llvm.org/git/clang-tools-extr\
a.git extra
cd $LLVM_ROOT/llvm/projects
git clone --depth=1 https://llvm.org/git/libcxx.git
git clone --depth=1 https://llvm.org/git/libcxxabi.git
git clone --depth=1 https://llvm.org/git/compiler-rt.git
mkdir $LLVM_ROOT/build
cd $LLVM_ROOT/build
```

Chandler recommends and demonstrates ninja based build.

Before invoking ninja based build, make sure the following packages are already installed.

```bash
sudo apt-get install -y cmake
sudo apt-get install -y cmake-curses-gui
sudo apt-get install -y ninja-build

# command to instruct ccmake to create  ninja build files.
ccmake -G Ninja $ROOT/llvm
```

This opens up ninja configuration console.
Press 'c' to configure.
Chandler recommends the following changes to the build configuration.
Change
* install path - CMAKE_INSTALL_PREFIX "$ROOT/install/`date`",
* release build not debug build - CMAKE_BUILD_TYPE "Rel\
ease",
* disable assertion - LLVM_ENABLE_ASSERTIONS - gives a \
 fast compiler

# c to configure; g to generate build files and exit

```bash
# build and install
ninja
ninja install
```

Also there is a recommendation to use a tool called lndir
that creates softlinks for all the files in the installation folder of llvm to the current folder.

```bash
sudo apt-get install -y xutils-dev # to install lndir
cd ~ # change to $HOME
lndir $ROOT/install/`date`
export PATH=$HOME/bin:$PATH
clang++ --version
```
Next Chandler demonstrates some c++17 features supported by the llvm toolchain thus built and setup.
* block initialization
* structured binding
* using string_view
  * string_view may not be available with standard c++ library that comes with the system (libstdc++).
  * Add `-stdlib=libc++` to compile command to direct clang to use libstdc++ that's built with llvm toolchain.
  * Chandler here emphasises that we have just not build clang the compiler but the entire toolchain.
For example, a different standard library can be chosen if required.
* clang-tidy
  * Chandler shows a loop that explicitly goes over a vector to illustrate how clang-tidy can help to transform the loop to a range based loop.
  * However, he missed to use -checks='*' when listing checks with -list-checks. So he could not successfully search for the "modernize-loop-convert" check.
  * So, here is the command to search for the relevant check and to use the check to tidy-up the code.
```bash
clang-tidy -list-checks -checks='*' | grep -i modern
clang-tidy -checks='modernize-loop-convert' -fix -range-loop.cpp -- -std=c++17
```
* sanitizers
  * asan - the address sanitizer
  * Address sanitizer could detect following class of bugs
    * use after free
    * use after scope
    * heap buffer overflows

  * add `-fsanitize=address` to compile command to compile the binary with address-sanitizer checks.

Here Chandler compares and contrasts clang sanitzers with valgrind's capabilities.
     * valgrind cannot detect stack buffer overflow but only heap buffer overflow and also has less precision.
     * valgrind has limited visibility into life time of objects (cannot catch use after scope)
     * valgrind can handle uninitialized memory errors. Address santizers can't handle it. But memory sanitizers can handle it.

 * To build the binary with memory sanitizer(msan) checks, add
   `-fsanitize=memory` to the compilation command
 * Using memory sanitizers on large projects requires all the files in the large project including the standard libraries used by the project be built with msan. If that's impossible, then valgrind is the only feasible option.

* tsan: thread sanitizer
  usage: `-fsanithize=thread`

* tsan gives a very conservative model of c++ memory model. Every time the strict semantics that the language guarantees is violated, tsan will report it.

* thinlto:
there are traditional techniques for link time optimization. windows has link time code generation. They usually 
      * have scaling limitation
      * are slow, challenging to deploy as large application

* to use thinlto, add the switch `-flto=thin`
  * The linker on the system may not have thinlto technology incorporated. But what has been built is just not the compiler clang but an entire toolchain. So we have lld, the linker which has thinlto implementation.
  * to use it, pass `-fuse-ld=lld`
  * So, the required switches are `-flto=thin -fuse-ld=lld`

To illustrate the advantage of employing thinlto, Chandler uses google benchmark to measure the performance of the application built with and without thinlto.
Let's first install google benchmark
Read the readme here without fail.
https://github.com/google/benchmark
and build as follows (taken from the readme).
```bash
git clone https://github.com/google/benchmark.git
git clone https://github.com/google/googletest.git benchmark/googletest
mkdir build && cd build

# replace Ninja by your favourite build files generator
# configure installation path and whatever is relevant
ccmake -G Ninja ../benchmark/

# use make if you used makefile generator
ninja
ninja install
```

Then build the program that use google benchmark as follows
```bash
export BENCH=/path/to/benchmark/installation
clang++ demo.1.cpp f.cpp -o demo.1.slow -O3 -std=c++17 -I$BENCH/include -L$BENCH/lib -lbenchmark -pthreads

# Now build with thin-lto
clang++ demo.1.cpp f.cpp -o demo.1.fast -O3 -std=c++17 -I$BENCH/include -L$BENCH/lib -lbenchmark -pthreads -flto=thin -fuse-ld=lld
```
One notable point is that, the advantage of using thin-lto manifests only when compiling with -O3.

-lto=thin can be used in production
chromium browser, MAC operating system are built with thi
n-lto.

Next demonstration is about thin-lto's part in checking control flow integrity (cfi)

A modern exploit is illustrated in cfi.h and cfi_call.cpp
The exploit does not show upwith typical way of build and run.

```bash
clang++ cfi_call.cpp -o cfi_call -std=c++17
```

But when `-fsanitize=cfi` is used, exploit leads to crash. The application crashes instead of giving contol to a wrong function.
```bash
clang++ cfi_call.cpp -o cfi_call -std=c++17 -fsanitize=cfi -flto=thin -fvisibility=hidden -fuse-ld=lld
```bash
cfi-sanitizer can be used only with -flto and -fvisibility.
The compiler gives nice help message which leads to using these options.

To get a nice error message instead of code dump, add `-fno-sanitize-trap=cfi`, So the command becomes
```bash
clang++ cfi_call.cpp -o cfi_call -std=c++17 -fsanitize=cfi -flto=thin -fvisibility=hidden -fuse-ld=lld -fno-sanitize-trap=cfi
```
The error message displayed is
cfi_call.cpp:37:9: runtime error: control flow integrity check for type 'DerivedB' failed during cast to unrelated type (vtable address 0x000000207ed0)
0x000000207ed0: note: vtable is of type 'DerivedA'

The error corresponds to the code
callB(reinterpret_cast<DerivedB &>(a));

where the prototype of callB is
void callB(DerivedB &b);

This build that includes control flow integrity can be shipped to production. Its designed to be very minimal and precise. Its a hardening/mitigation technique

Watch the talk here.
https://www.youtube.com/watch?v=uZI_Qla4pNA
