---
title: "Emscripten with CMake"
date: 2022-07-16T15:30:05-06:00
draft: false
tags: ['emscripten','cmake','c++','webgl']
---

TLDR: If you just want a [quick solution](#tldr).

Emscripten is a pretty powerful tool for porting C and C++ to JavaScript. It does this through clang which can compile to WebAssembly, Wasm, and it's own suite of libraries for making things "just work". It's fairly straightforward and mostly a drop in replacement for your standard compiler with some exceptions (eg: threads). Dropping it into an existing CMake project, however, isn't as straightforward. I ran into a few quirks that I had trouble finding documented processes online.

There is a working example of a library that compiles both x86_64 and wasm below, along with some notes that you may find helpful.



## Prerequisites

Make sure you install emscripten first, installation is platform specific. Once you have it installed you will end up with some emscripten folder that you will need to reference for your CMake toolchain or you will need to trigger the `emcmake` helper instead.

## Example Library

Let's take an existing static or shared library along with a test executable and convert it so it works for the web.

### calc.cpp
~~~C++
extern "C" 
{
    int add(int a, int b)
    {
        return a + b;
    }
    int sub(int a, int b)
    {
        return a - b;
    }
}
~~~

### calc.hpp
~~~C++
extern "C" 
{
    int add(int a, int b);
    int sub(int a, int b);
}
~~~

### test.cpp
~~~C++
#include "calc.hpp"
#include <iostream>

int main()
{
    int a = add(1,2);
    int b = sub(2,1);
    
    std::cout << "1+2 => " << a << std::endl;
    std::cout << "2-1 => " << b << std::endl;
    
    if(a != 3)
    {
        std::cerr << "Invalid add" << std::endl;
        return -1;
    }
    if(b != 1)
    {
        std::cerr << "Invalid sub" << std::endl;
        return -1;
    }

    return 0;
}
~~~

### CMakeLists.txt
~~~CMake
cmake_minimum_required(VERSION 3.8)
project(test)
add_definitions(-std=c++17)
set (CMAKE_CXX_STANDARD 17)
#set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS 1) # windows/msvc needs explicit "export all"
add_library(calc SHARED calc.cpp)
add_executable(test test.cpp)
target_link_libraries(test PUBLIC calc)
~~~

### Building and Running

Now we'll build and verify our test executable and calc library exist and work properly

```
mkdir build
cd build

cmake cmake ..
cmake --build . --target calc
cmake --build . --target test
```

This will produce both a test executable and a libcalc.so (on linux). Let's make sure our symbols for `add` and `sub` are exported properly with `objdump`

```
objdump -TC libcalc.so

libcalc.so:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
0000000000000000  w   DF *UND*  0000000000000000  GLIBC_2.2.5 __cxa_finalize
0000000000000000  w   D  *UND*  0000000000000000              _ITM_deregisterTMCloneTable
0000000000000000  w   D  *UND*  0000000000000000              __gmon_start__
0000000000000000  w   D  *UND*  0000000000000000              _ITM_registerTMCloneTable
0000000000001100 g    DF .text  0000000000000012  Base        add
0000000000001120 g    DF .text  0000000000000012  Base        sub
```

Great we can see both `add` and `sub` in the global symbols and they are defined.

Now let's run our binary `test` and check the results.

```
./test
1+2 => 3
2-1 => 1
```

Perfect we have our working executable and library we want to port.

### Porting

#### Invoking Emscripten with CMake

We can either call `emcmake` which will wrap our cmake call with the toolchain file or we can directly specify it with cmake.

If you want to specify `-DCMAKE_TOOLCHAIN_FILE` yourself, you will need to locate your `Emscripten.cmake` file which is included with emscripten. In my case I set a PATH variable `EMROOT` so it points to the emscripten installation folder and then can invoke cmake like so:

```
cmake -DCMAKE_TOOLCHAIN_FILE=$EMROOT/cmake/Modules/Platform/Emscripten.cmake ..
```

Our other option (the easier one) is to just add `emcmake` before our cmake call like so:

```
emcmake cmake ..
```

We can then just call our normal make/cmake calls without wrapping them.

```
cmake --build . --target calc
cmake --build . --target test
```

Let's see what happens if we just stop here, does this work?

```
mkdir build.em
cd build.em

emcmake cmake ..
cmake --build . --target calc
cmake --build . --target test
```

This produces a few files

1. libcalc.a
2. test.js
3. test.wasm

We'll make a simple web page and host it pointing to the test.js that cmake and emscripten made for us.

#### index.html
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Simple template</title>
  </head>
  <body>
      <script type="text/javascript" src="build.em/test.js"></script>
  </body>
</html>
~~~

If we point to this page and load it, checking the developer console we can indeed see output:

```
test.js:2128 1+2 => 3
test.js:2128 2-1 => 1
```

We can also see that we can call `Module._main()` from the console which will re-run our program:

```
Module._main()
test.js:2128 1+2 => 3
test.js:2128 2-1 => 1
0
```

Neat, well that would be where we'd stop if we just want to run an executable, but we want to be able to use the calc library and call our `add` and `sub` functions ourselves, so how do we do that? We don't see a `Module._add` or similar functions in the object.

This is where it gets a bit harder, that test.js file was produced for us and handles all of the instantiation for us.

Let's take a look at that `libcalc.a` archive file. If we open it up with `ar x libcalc.a` it will produec a `calc.cpp.o` object file but it's really a wasm file. We can inspect it using `wasm-objdump` (see below):

```
ar x libcalc.a
wasm-objdump -x calc.cpp.o

calc.cpp.o:     file format wasm 0x1

Section Details:

Type[1]:
 - type[0] (i32, i32) -> i32
Import[2]:
 - memory[0] pages: initial=0 <- env.__linear_memory
 - global[0] i32 mutable=1 <- env.__stack_pointer
Function[2]:
 - func[0] sig=0 <add>
 - func[1] sig=0 <sub>
Code[2]:
 - func[0] size=61 <add>
 - func[1] size=61 <sub>
Custom:
 - name: "linking"
  - symbol table [count=3]
   - 0: F <add> func=0 binding=global vis=default
   - 1: G <env.__stack_pointer> global=0 undefined binding=global vis=default
   - 2: F <sub> func=1 binding=global vis=default
Custom:
 - name: "reloc.CODE"
  - relocations for section: 3 (Code) [2]
   - R_WASM_GLOBAL_INDEX_LEB offset=0x000006(file=0x00005f) symbol=1 <env.__stack_pointer>
   - R_WASM_GLOBAL_INDEX_LEB offset=0x000044(file=0x00009d) symbol=1 <env.__stack_pointer>
Custom:
 - name: "producers"
```

It does indeed expose the add and sub functions. There is a problem though in that in the `Import` section it needs `__linear_memory` and `__stack_pointer`. This will be a problem but we can define these when loading directly through WebAssembly API. There is one other concern which is that the `Export` section doesn't provide any definitions. Let's try to load our wasm file directly though, by creating a new html file. We'll copy calc.cpp.o to calc.wasm, then produce this stub html.

#### wasm.html
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Simple template</title>
  </head>
  <body>
    <script>
      const importObject = {
        env: {
          __linear_memory: new WebAssembly.Memory({ initial: 256 }),
          __stack_pointer: new WebAssembly.Global({value:'i32', mutable:true,}, 0),
        }
      };

      WebAssembly.instantiateStreaming(
        fetch('build.em/calc.wasm'),
        importObject
      ).then(result => {
        const add = result.instance.exports.add;
        const sub = result.instance.exports.sub;
        console.log(add(1,2));
        console.log(sub(2,1));
      });
    </script>
  </body>
</html>
~~~

Running this in a browser we get this error in the console:

```
wasm.html:24 Uncaught (in promise) TypeError: add is not a function
    at wasm.html:24:21
```

Looks like we need to export these functions in some way, at this point we're going to need to modify our `CMakeLists.txt` now.



#### Modifying Your CMakeLists.txt

The easiest way of making this work is converting your `add_library` call to an `add_executable`. Let's try that now and take a look at what it produces.

We'll change

~~~CMake
add_library(calc SHARED calc.cpp)
~~~

to

~~~CMake
add_executable(calc calc.cpp)
~~~

Since we no longer need the test app either we can comment that out for now `#add_executable(test test.cpp)`. After rebuilding we'll see both a `calc.js` and a `calc.wasm` file. Let's take a look at the wasm file first.

```
wasm-objdump -x calc.wasm

calc.wasm:      file format wasm 0x1

Section Details:

Type[7]:
 - type[0] () -> i32
 - type[1] (i32) -> nil
 - type[2] () -> nil
 - type[3] (i32) -> i32
 - type[4] (i32, i32) -> i32
 - type[5] (i32, i32, i32) -> i32
 - type[6] (i32, i64, i32) -> i64
Function[18]:
 - func[0] sig=2 <__wasm_call_ctors>
 - func[1] sig=4 <add>
 - func[2] sig=4 <sub>
 - func[3] sig=0 <stackSave>
 - func[4] sig=1 <stackRestore>
 - func[5] sig=3 <stackAlloc>
 - func[6] sig=2 <emscripten_stack_init>
 - func[7] sig=0 <emscripten_stack_get_free>
 - func[8] sig=0 <emscripten_stack_get_base>
 - func[9] sig=0 <emscripten_stack_get_end>
 - func[10] sig=1
 - func[11] sig=1
 - func[12] sig=0
 - func[13] sig=2
 - func[14] sig=3
 - func[15] sig=1
 - func[16] sig=3 <fflush>
 - func[17] sig=0 <__errno_location>
Table[1]:
 - table[0] type=funcref initial=1 max=1
Memory[1]:
 - memory[0] pages: initial=256 max=256
Global[3]:
 - global[0] i32 mutable=1 - init i32=5243920
 - global[1] i32 mutable=1 - init i32=0
 - global[2] i32 mutable=1 - init i32=0
Export[14]:
 - memory[0] -> "memory"
 - func[0] <__wasm_call_ctors> -> "__wasm_call_ctors"
 - func[1] <add> -> "add"
 - func[2] <sub> -> "sub"
 - table[0] -> "__indirect_function_table"
 - func[17] <__errno_location> -> "__errno_location"
 - func[16] <fflush> -> "fflush"
 - func[6] <emscripten_stack_init> -> "emscripten_stack_init"
 - func[7] <emscripten_stack_get_free> -> "emscripten_stack_get_free"
 - func[8] <emscripten_stack_get_base> -> "emscripten_stack_get_base"
 - func[9] <emscripten_stack_get_end> -> "emscripten_stack_get_end"
 - func[3] <stackSave> -> "stackSave"
 - func[4] <stackRestore> -> "stackRestore"
 - func[5] <stackAlloc> -> "stackAlloc"
Code[18]:
 - func[0] size=4 <__wasm_call_ctors>
 - func[1] size=57 <add>
 - func[2] size=57 <sub>
 - func[3] size=4 <stackSave>
 - func[4] size=6 <stackRestore>
 - func[5] size=18 <stackAlloc>
 - func[6] size=20 <emscripten_stack_init>
 - func[7] size=7 <emscripten_stack_get_free>
 - func[8] size=4 <emscripten_stack_get_base>
 - func[9] size=4 <emscripten_stack_get_end>
 - func[10] size=2
 - func[11] size=2
 - func[12] size=10
 - func[13] size=7
 - func[14] size=4
 - func[15] size=2
 - func[16] size=304 <fflush>
 - func[17] size=5 <__errno_location>
 ```
 
That's a ton of extra stuff from before, but the important parts are both `add` and `sub` are both in the `Function` and `Export` section. Let's try this in our html file now and see if we can get this to work. Since we produced both a `.js` and `.wasm` file we can reference either. 

### Option 1 - Referencing the wasm directly

Since we no longer have required `Import` sections, we can remove our environment importObject options.

#### wasm.html
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Simple template</title>
  </head>
  <body>
    <script>
      WebAssembly.instantiateStreaming(
        fetch('build.em/calc.wasm'),
      ).then(result => {
        const add = result.instance.exports.add;
        const sub = result.instance.exports.sub;
        console.log(add(1,2));
        console.log(sub(2,1));
      });
    </script>
  </body>
</html>
~~~

Looking at our console log, it produces the following output:

```
wasm.html:24 3
wasm.html:25 1
```

Now we have a working library we can call from our browser.

###  Option 2 - Referencing the js directly

Since we also produced a JavaScript file, we can reference that instead. However we'll need to change how we invoke our functions. We'll be calling into the `Module` object, in addition we'll need to wait for Module to finish loading as well. Note the necessary "_" underscore prefix here.

#### wasm.html
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Simple template</title>
  </head>
  <body>
      <script type="text/javascript" src="build.em/calc.js"></script>
      <script>
        Module.onRuntimeInitialized = function()
        {
            console.log(Module._add(1,2));
            console.log(Module._sub(2,1));
        };
      </script>
  </body>
</html>
~~~

This also produces the correct output:

```
(wasm.html):12 3
(wasm.html):13 1
```

There are several options you can take on loading/deferring/etc your scripts. My personal preference is to reference the `wasm` file directly via the `WebAssembly.instantiateStreaming` call since I have complete control over success or failure states and order guarantees.

### Optimizing

While our working example does indeed work, there are many options we can pass in to adjust what get's built. By default our wasm file is built with `-O0` meaning no optimizations. Let's drop in a few arguments to help reduce unnecessary exports and optimizations now.

#### CMakeLists.txt
~~~CMake
cmake_minimum_required(VERSION 3.8)

project(test)

add_definitions(-std=c++17)
set (CMAKE_CXX_STANDARD 17)

if (DEFINED EMSCRIPTEN)
	add_executable(calc calc.cpp calc.hpp)

	set(CMAKE_EXECUTABLE_SUFFIX ".wasm")

	set_target_properties(calc PROPERTIES COMPILE_FLAGS "-Os -s SIDE_MODULE=1 ")
	set_target_properties(calc PROPERTIES LINK_FLAGS    "-Os -s WASM=1 -s SIDE_MODULE=1 -s STANDALONE_WASM --no-entry")
else()
	add_library(calc SHARED calc.cpp)
endif()
~~~

Let's look at the options we set to explain what they're doing

1. `-Os`  
    Specifies that we want optimizations and code size reductions (very useful for web). These are pretty much the same things as traditional optimization flags in gcc/clang.
2. `-s WASM=1`  
    This is the default value, but doesn't hurt to set. Values can be 0, 1, or 2. Where 0 means compile to JavaScript, 1 means compile to WebAssembly, and 2 means compile to both.
3. `-s SIDE_MODULE=1`  
    If we set this to 2 we'll remove functions that don't have `EMSCRIPTEN_KEEPALIVE` macros in their definitions (ie: calc.cpp). This defaults to 0, by specifying 1 here it removes many additional entries in the wasm library that we don't currently need.
4. `-s STANDALONE_WASM`  
    Specifies that we want to build a WebAssembly file that doesn't need a supporting JavaScript file to run. Normally we would need a main entry point, so we also include the `--no-entry` flag as well.
5. `--no-entry`  
    Specifies that we're not using a main entry point in our code (ie: no `int main(...)`).
6. `set(CMAKE_EXECUTABLE_SUFFIX ".wasm")`  
    Tells CMake and Emscripten to produce a wasm file instead of defaulting to a javascript one



## TLDR

The easiest solution is to convert your `add_library` call to an `add_executable` call, this will produce a `.js` or `.wasm` file that you can directly load in a browser.

## Minimum Working Example

In summary if you want to reference the working example files, see below:

### calc.cpp
~~~C++
#ifdef __EMSCRIPTEN__
#include <emscripten/emscripten.h>
#else
#define EMSCRIPTEN_KEEPALIVE
#endif

extern "C" 
{
    EMSCRIPTEN_KEEPALIVE int add(int a, int b)
    {
        return a + b;
    }
    EMSCRIPTEN_KEEPALIVE int sub(int a, int b)
    {
        return a - b;
    }
}
~~~

### calc.hpp
~~~C++
extern "C" 
{
    int add(int a, int b);
    int sub(int a, int b);
}
~~~

### CMakeLists.txt
~~~CMake
cmake_minimum_required(VERSION 3.8)

project(test)

add_definitions(-std=c++17)
set (CMAKE_CXX_STANDARD 17)

if (DEFINED EMSCRIPTEN)
	add_executable(calc calc.cpp calc.hpp)

	set(CMAKE_EXECUTABLE_SUFFIX ".wasm")

	set_target_properties(calc PROPERTIES COMPILE_FLAGS "-Os -s SIDE_MODULE=1 ")
	set_target_properties(calc PROPERTIES LINK_FLAGS    "-Os -s WASM=1 -s SIDE_MODULE=1 -s STANDALONE_WASM --no-entry")
else()
	add_library(calc SHARED calc.cpp)
endif()
~~~

### index.html
~~~html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Calculator Test</title>
  </head>
  <body>
    <script>
      WebAssembly.instantiateStreaming(
        fetch('build.em/calc.wasm'),
      ).then(result => {
        const add = result.instance.exports.add;
        const sub = result.instance.exports.sub;
        console.log(add(1,2));
        console.log(sub(2,1));
      });
    </script>
  </body>
</html>
~~~

### build.sh
A simple build script from the CLI
~~~bash
#!/bin/bash
# Either set EMROOT to your emscripten root folder, or ensure emscripten is setup in your path
# You can trip the emsdk/emsdk_env.sh via or set in your .bash_profile or similar
#  source "...path_to_emscripten.../emsdk/emsdk_env.sh"

#cmake -DCMAKE_TOOLCHAIN_FILE=$EMROOT/cmake/Modules/Platform/Emscripten.cmake ..
emcmake cmake ..
cmake --build . --target calc
~~~



## Notes and Quirks

### Without CMake or Emscripten

If you want to compile without CMake, it's quite simple, we can build via:

```
mkdir build
cd build

clang++ -Wall -shared ../calc.cpp -o libcalc.so
clang++ -Wall ../test.cpp -I ../calc.hpp ./libcalc.so -o test
```

This will produce both a libcalc.so and test files. We can then make sure it's linking properly via `ldd test`

```
ldd test
        linux-vdso.so.1 (0x00007ffe3c874000)
        ./libcalc.so (0x00007fc4dda17000)
        libstdc++.so.6 => /lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fc4dd82c000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fc4dd6dd000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fc4dd6c2000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc4dd4d0000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fc4dda1e000)
```

And running `./test` produces the result we're looking for

```
./test
1+2 => 3
2-1 => 1
```

### Additional Topics

If you are looking to better fine tune your results, here are some topics that you'll want to look at.

1. ccal, cwrap, EXPORTED_RUNTIME_METHODS  
    https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html#interacting-with-code-ccall-cwrap
2. lld WebAssembly Options - explains things like `--no-entry`, `--export-all` and other options  
    https://lld.llvm.org/WebAssembly.html
3. emcc options  
    https://emscripten.org/docs/tools_reference/emcc.html
4. emcc settings.json - explain this like `SIDE_MODULE` vs `MAIN_MODULE` and other options that can be used at compile and link time  
    https://github.com/emscripten-core/emscripten/blob/main/src/settings.js
5. emscripten api - programming interace options such as `EMSCRIPTEN_KEEPALIVE`
    https://emscripten.org/docs/api_reference/emscripten.h.html

### Additional Resources

1. https://surma.dev/things/c-to-webassembly/
2. https://users.rust-lang.org/t/wasm-unknown-vs-emscripten/22997/4
3. https://stackoverflow.com/questions/64690937/what-is-the-difference-between-emscripten-and-clang-in-terms-of-webassembly-comp
4. https://depth-first.com/articles/2019/10/16/compiling-c-to-webassembly-and-running-it-without-emscripten/
5. https://github.com/WebAssembly/wasi-sdk
6. https://reviews.llvm.org/D34172?id=102407
7. https://blog.meain.io/2017/ehh-webassembly/

### C Interface Wrapping

Just FYI, we want to wrap any C++ functions with `extern "C"` to prevent name mangling. This is common in many C++ libraries which provide a C interface anyway and it is quite common in bespoke libraries.

### EMSCRIPTEN_KEEPALIVE

This eliminates the need to specify the functions you want to export in your flags when `MAIN_MODULE` or `SIDE_MODULE` is set at a value or 2. This is very helpful if you want to remove other code in your library that you don't plan to expose for Wasm.

For example we could modify our calc.cpp file as follows:

~~~C++
#ifdef __EMSCRIPTEN__
#include <emscripten/emscripten.h>
#else
#define EMSCRIPTEN_KEEPALIVE
#endif

extern "C" 
{
    EMSCRIPTEN_KEEPALIVE int add(int a, int b)
    {
        return a + b;
    }
    EMSCRIPTEN_KEEPALIVE int sub(int a, int b)
    {
        return a - b;
    }
}
~~~

### Library Building

When building a library via `add_library` your wasm will be inside an archive file, you can extract it via `ar -x your_file.a` which will extact the wasm files with `.o` object suffixes. You can then just rename or load these directly if you wish. If you want to skip this process you can just cheat and swap your `add_library` into an `add_executable`, this is easily done with a if check for `if (DEFINED EMSCRIPTEN)` and branch in your cmake.

### Debugging

Install the `wabt` tools, this gives you access to tools like `wasm-objdump` which are very helpful in debugging which functions you've exported and provided code for.

Running objdump against our calc library wasm


```
wasm-objdump -x calc.wasm

calc.wasm:      file format wasm 0x1

Section Details:

Custom:
 - name: "dylink.0"
Type[2]:
 - type[0] (i32, i32) -> i32
 - type[1] () -> nil
Function[3]:
 - func[0] sig=1 <__wasm_apply_data_relocs>
 - func[1] sig=0 <add>
 - func[2] sig=0 <sub>
Export[4]:
 - func[0] <__wasm_apply_data_relocs> -> "__wasm_call_ctors"
 - func[0] <__wasm_apply_data_relocs> -> "__wasm_apply_data_relocs"
 - func[1] <add> -> "add"
 - func[2] <sub> -> "sub"
Code[3]:
 - func[0] size=3 <__wasm_apply_data_relocs>
 - func[1] size=7 <add>
 - func[2] size=7 <sub>
```

 You can clearly see the `add` and `sub` functions exported and defined in the wasm.
