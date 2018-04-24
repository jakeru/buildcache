# BuildCache

This is a simple compiler accelerator that caches and reuses build results to
avoid unnecessary re-compilations, and thereby speeding up the build process.

It is similar in spirit to [ccache](https://ccache.samba.org/).

## Building

Use [CMake](https://cmake.org/) and your favorite C++ compiler to build the BuildCache program:

```bash
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release ../src
$ cmake --build .
```

## Usage

To use BuildCache for your builds, simply prefix the build command with
`buildcache`. For instance:

```bash
$ buildcache g++ -c -O2 hello.cpp -o hello.o
```

A convenient solution for bigger CMake-based projects is to use the
`RULE_LAUNCH_COMPILE` property to use BuildCache for all compilation commands,
like so:

```cmake
find_program(buildcache_program buildcache)
if(buildcache_program)
  set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${buildcache_program}")
endif()
```

## Using with icecream

[icecream](https://github.com/icecc/icecream) (or ICECC) is a tool for
distributed compilation. To use icecream you can set the environment variable
`BUILDCACHE_PREFIX` to the icecc executable, e.g:

```bash
$ BUILDCACHE_PREFIX=/usr/bin/icecc buildcache g++ -c -O2 hello.cpp -o hello.o
```

Note: At the time of writing there is a [bug](https://github.com/icecc/icecream/issues/390)
in ICECC that may disable distributed compilation when ICECC is invoked via BuildCache.

## Supported compilers and languages

Currently the following compilers and languages are supported:

| Compiler | Languages | Supported |
| -------- | --------- | --------- |
| GCC      | C, C++    | Yes       |
| Clang    | C, C++    | Yes       |
| MSVC     | C, C++    | Yes       |
| GHS      | C, C++    | Yes       |

New backends are relatively easy to add, both in C++ and in Lua (see below).

## Using custom Lua plugins

It is possible to extend the capabilities of BuildCache with [Lua](https://www.lua.org/).

BuildCache first searches for Lua scripts in the paths given in the environment variable `BUILDCACHE_LUA_PATH` (colon separated on POSIX systems, and semicolon separated on Windows), and then continues searching in `$BUILDCACHE_DIR/lua`. If no matching script file was found, BuildCache falls back to the built in compiler wrappers (as listed above).

**Note:** To use Lua standard libraries (`coroutine`, `debug`, `io`, `math`, `os`, `package`, `string`, `table` or `utf8`), you must first load them by calling `require_std(name)`. For convenience it is possible to load all standard libraries with `require_std("*")`, but beware that it is slower than to load only the libraries that are actually used.

All program arguments are available in the global `ARGS` array (an array of strings).

Here is a minimal Lua example that caches the output of the "echo" command (yes, it's fairly pointless).

**echo.lua**
```lua
require_std("string")

function can_handle_command (compiler_exe)
  -- Is the "echo" command being invoked?
  return compiler_exe:lower():find("echo") ~= nil
end

function preprocess_source ()
  -- We do not generate a "preprocessed source".
  return ''
end

function get_relevant_arguments ()
  -- Return the arguments that may affect the result (i.e. all arguments).
  return ARGS
end

function get_relevant_env_vars ()
  -- There are no environment variables that affect the program result.
  return {}
end

function get_program_id ()
  -- We use the full path to the executable as a program identifier.
  return ARGS[0]
end

function get_build_files ()
  -- This command will not produce any output files to be cached.
  return {}
end
```

The following methods can be implemented (see [program_wrapper.hpp](src/program_wrapper.hpp) for a more detailed documentation):

| Function | Returns | Default |
| --- | --- | --- |
| can_handle_command (program_exe) | Can the wrapper handle this program? | - |
| preprocess_source () | The preprocessed source code (e.g. for C/C++) | An empty string |
| get_relevant_arguments () | Arguments that can affect the build output | All arguments |
| get_relevant_env_vars () | Environment variables that can affect the build output | An empty table |
| get_program_id () | A unique program identification | The MD4 hash of the binary |
| get_build_files () | A table of build result files | An empty table |

## Debugging

To get debug output from a BuildCache run, set the environment variable
`BUILDCACHE_DEBUG` to the desired debug level:

| BUILDCACHE_DEBUG | Level | Comment           |
| ---------------- | ----- | ----------------- |
| 1                | DEBUG | Maximum printouts |
| 2                | INFO  |                   |
| 3                | ERROR |                   |
| 4                | FATAL |                   |

For instance:

```bash
$ BUILDCACHE_DEBUG=2 buildcache g++ -c -O2 hello.cpp -o hello.o
```

## Status

**NOTE:** BuildCache is still in early development and should not be considered
ready for production projects yet!

