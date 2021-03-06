# Catalog format specification

## Big picture

Each release belonging to a crate is described as a TOML file. This file has
minor differences depending on its location: a local manifest is found at the
top-level directory of the sources of a project, in which case its named
`alire.toml`, whereas a manifest found in an index (e.g., the community index),
is named `<name>-<version>.toml`.

Other than that, contents follow the same conventions and there are only slight
differences (some fields are intended only for an index manifest, and cannot
appear, or are optional, in a local manifest). These differences are highlighted
in the following descriptions, where necessary.

Each TOML description file contains exactly one release, except for the special
external definitions that are described in their own section. 

## Information encoding

This section describes the various encodings used in this format to encode
information.

First, there are two kinds of data: atomic and composite.

Atomic data designates values that cannot be decomposed. There are only two
atomic data types:

 - mere strings (`"Hello, world!"`);
 - booleans (`true`, `false`);

We can then split composite data in two kinds: lists (TOML's arrays) and
mappings (JSON's tables). Lists are just sequences of other values, for
instance a list of strings:

```toml
["A", "B"]
```

Mappings are the traditional sets of associations from keys (here, always
strings) to other values. For instance, the following represents a set of
dependencies, with version constraints:

```toml
libfoo = "^1.2"
libbar = "^2.0 & /=2.1.3" # Excluding a known bad version
```

In some contexts, information can be dynamic: special encodings can be used to
make data vary depending on the environment (OS, architecture, ...). The
environment is represented as a set of specific variables which can have a
specific set of values: see the [Parameters](#parameters) section below for a
comprehensive list.

All properties that support dynamic expressions follow the same structure, in
which the expression (case-like) is inserted between the key and its value.
For example, given a static expression:

```toml
key = "value"
```

one of its cases would be expressed by the following inline TOML table:

```toml
key.'case(var)'.var_value = "value"
```

Several expressions can be inserted between a property key and its value,
leading to a combinatorial explosion if all cases have specific values. The
equivalent to an `others` Ada clause in this format is a `'...'` entry.

Here is an example of a conditional boolean value.

```toml
{'case(distribution)' = {
    'debian|ubuntu': true,
    '...': false
}}

# Or in a more idiomatic TOML syntax
['case(distribution)']
'debian|ubuntu' = true
'...' = false
```

Depending on the value of the `distribution` environment variable, this will
return `true` (its value is `debian` or `ubuntu`) or `false` (for other
values). Note that these and subsequent examples are not showing the
left-hand-side property to which such a value would be assigned.

A little variation allows building environment-dependent composite data. For
instance, to make the dependency on `libbar` above dynamic:

```toml
{
    "libfoo": "^1.2",
    "case(os)": {
        "linux": {"libbar": "^2.0"},
        "windows": {"libwinbar": "^3.0"},
        "...": {}
    }
}

# Or in a more idiomatic TOML syntax
libfoo = "^1.2"

['case(os)'.linux]
libbar = "^2.0"

['case(os)'.windows]
libwinbar = "^3.0"

['case(os)'.'...']
```

The `case(os)` part selects dependencies depending on the value of the `os`
environment variable.

If the `os` environment variable contains `linux`, this will create the
following dependencies:

```toml
libfoo = "^1.2"
libbar = "^2.0"
```

If the `os` environment variable contains `windows`, this will create the
following dependencies:

```toml
libfoo = "^1.2"
libwinbar = "^3.0"
```

And finally for other `os` values:

```toml
libfoo = "^1.2"
```

## Release Information

This section describes the actual properties that must or can appear in a
manifest file to describe a release. Unless specified, all the entries must be
static, i.e. they cannot depend on the context.

 - `name`: mandatory string. The name of the crate this release belongs to. Use
   `alr help identifiers` to see the rules such names must follow.

 - `version`: mandatory string. The semantic version of the release.

 - `description`: mandatory string. One-line description about the package. For
   instance:

   ```toml
   description = "Library to handle foobars"
   ```

 - `long-description`: optional free-form string to provide information about
   this package, in addition to `description`, without length restrictions.

 - `authors`: optional array of strings. Flat list of human-readable names for
   the authors, i.e. the people that wrote the software that is packaged. For
   instance:

   ```toml
   authors = ["Alice Example",
              "Bob For Instance <bob@example.com>"]
   ```

 - `maintainers`: mandatory (for indexing) array of strings. Flat list of
   human-readable names (optional) for the maintainers, with a contact email
   (mandatory); i.e. the people that maintain the crate metadata in Alire. For
   instance:

   ```toml
   maintainers = ["alice@example.com",
                  "Bob For Instance <bob@athome.com>"]
   ```

 - `maintainers-logins`: mandatory (for indexing) array of strings. Flat
   list of github login usernames used by the maintainers of the crate. This
   information is used to authorize crate modifications. For instance:

   ```toml
   maintainers-logins = ["alicehacks", "bobcoder"]
   ```

 - `licenses`: mandatory (for indexing) string. A valid [SPDX
   expression](https://spdx.org/licenses/). Custom license identifiers are
   accepted with the format: `custom-[0-9a-zA-Z.-]+`

   ```toml
   licenses = "MIT"
   ```

   For a double license:

   ```toml
   licenses = "GPL-3.0-only OR MIT"
   ```

   For a custom license:

   ```toml
   licenses = "custom-my-license-1.2"
   ```

 - `website`: optional string. URL to the original project's website. For
   instance:

   ```toml
   website = "https://myproject.example.org/"
   ```

 - `tags`: optional array of strings. Flat list of topics covered by the crate.
   Tags will help users find crates related to their interests:

   ```toml
   tags = ["spark", "security"]
   ```

 - `available`: optional dynamic boolean expression. Determines whether the
   package is available for the current platform (true) or not (false). For
   instance:

   ```toml
   [available.'case(distribution)']
   'debian|ubuntu' = true
   '...' = false
   ```

 - `depends-on`: optional array of dynamic dependencies expressions. For instance:

   ```toml
   [[depends-on]]  # A static dependency
   libfoo = "^1.2"

   [[depends-on]]  # A dynamic dependency
   [depends-on.'case(os)'.linux]
   libbar = "^2.0"

   [depends-on.'case(os)'.windows]
   libwinbar = "^3.0"
   ```

   Available constraint operators are the usual Ada relationals (`=`, `/=`, `>`, `>=`,
   `<`, `<=`) plus caret (`^`, any upwards version within the same major point)
   and tilde (\~, any upwards version within the same minor point). 

   **Note that caret and tilde do not have any special behavior for pre-1
   versions.** This means, for example, that `^0.2` will still mean any release
   below `1.0`. The Semver specification does not make any promises about the
   compatibility of pre-1 versions, and there are differing interpretations of
   these operators out there for such versions. Bear in mind this when expressing
   your restrictions; for pre-1 versions you most likely want to use `~0.x`
   constraints (compatibility within a minor version).

   Logical operators for and (&), or (|) are accepted; see the `Semantic_Versioning`
   project documentation on [extended version
   sets](https://github.com/alire-project/semantic_versioning#types).

 - `project-files`: optional list of strings. Each is a path, relative to the
   root of the source directory, to a `.gpr` project file to be made available.
   Expressions are accepted. For instance:

   ```toml
   project-files = ["my_project.gpr", "utils/utils_for_my_project.gpr"]

   [project-files.'case(word-size)']
   bits-64 = ["my_project.gpr"]
   bits-32 = ["my_project32.gpr"]
   ```

 - `gpr-externals`: optional table, giving a mapping from the name of external
   variables in the `.gpr` project files to sets of possible values (as array of
   strings), or an empty string if this set is infinite. For instance:

   ```toml
   [gpr-externals]
   BUILD_MODE = ["debug", "profile", "release"]
   TAG = ""
   ```

 - `gpr-set-externals`: optional dynamic table, setting values of project 
   external variables when building the project. This should not be used to
   specify default values, the default values must be specified in the `.gpr`
   project file. Expressions are accepted before the mapping. For instance:

   ```toml
   [gpr-set-externals]
   BUILD_MODE = "release"

   [gpr-set-externals.'case(os)']
   linux   = { OS = "gnu-linux" } # Compact table syntax is convenient in this case
   windows = { OS = "ms-linux" }  # to see all enumeration values, one per row.
   ```

 - `environment`: optional dynamic table used to modify environment variables
   that will apply at build time. Variables and values are specified with the
   form `VARIABLE.<action> = "value"`, where `<action>` is one of `append`,
   `prepend`, or `set`. For instance:

   ```toml
   [environment]
   C_INCLUDE_PATH.append = "/usr/include/something"
   MY_PROJECT_ASSETS.set= "${CRATE_ROOT}/assets"
   PATH.append = "${DISTRIB_ROOT}/usr/bin"
   ```

   Predefined variables are provided by Alire and will be replaced in the
   value:

   - `${CRATE_ROOT}` absolute path to the deployment directory of the crate.
   - `${DISTRIB_ROOT}` absolute path to the root directory of the system
     distribution. On UNIX systems it will be `/`, on Windows `msys2` it will
     be the `msys2` installation directory (e.g.
     `C:\Users\user_name\.cache\alire\msys2`).

   Environment entries can use dynamic expressions:

   ```toml
   [environment.'case(distribution)']
   msys2 = { C_INCLUDE_PATH.append = "${DISTRIB_ROOT}/mingw64/include/SDL2" }
   ```

 - `executables`: optional dynamic list of strings. Each one is the simple name
   of an executable provided by the package. Executables are looked for by
   `alr` in the build tree and must not include a path. If only one executable is
   given, it is considered the default for `alr run`. For instance:

   ```toml
   executables = ["my_main"]
   ```

 - `actions`: optional dynamic list of actions to perform when installing this
   package.  The general action syntax is:

   ```toml
   [[actions]]
   type = <kind>
   command = <command>
   ```

   `<command>` is an array of strings for a shell command to run in the
   source directory. `<kind>` can be either:

   - `post-fetch`: the command is to be run right after getting the package
      sources. This action is run for all releases in a workspace.

   - `pre-build`: the command is to be run right before GPRbuild is run. This
      kind of action is run only for the root crate in a workspace.

   - `post-build`: the command is to be run right after GPRbuild has been
      run. This kind of action is run only for the root crate in a workspace.

   - `test`: the command is run on demand for crate testing within the Alire 
      ecosystem (using `alr test`). This kind of action is fun only for the
      root crate being tested.

   Actions accept dynamic expressions. For example:

   ```toml
   [[actions.'case(os)'.linux]]
   type = "post-fetch"
   command = ["make"]

   [[actions.'case(os)'.windows]]
   type = "post-fetch"
   command = ["cmd", "build"]

   [[actions.'case(os)'.'...']]
   # An explicit empty case alternative, which is not mandatory
   ```

 - `auto-gpr-with`: optional Boolean value that specifies if the project (gpr) files
   of a crate can be automatically depended upon ('withed') directly by the root
   project file. (The default is true.) This feature is meant to simplify the process
   of using dependencies in Alire. However, not all project files are supposed to be
   direct dependencies. Some are intended to be extended, for example, and in that
   case a crate can disable the feature by setting `auto-gpr-with=false`.

 - `origin`: dynamic table. Mandatory for index manifests and forbidden in
   workspace manifests. This table describes how sources are obtained, using
   the following fields:

      - `url`: mandatory string which points to a source file or repository.

      - `hashes`: mandatory string array for source archives.  An array
        of "kind:digest" fields that specify a hash kind and its value.  Kinds
        accepted are: `sha512`.

      - `archive-name`: optional string. If `url` points to a source archive,
        this can specify the name of the file to download, which is needed in
        order to properly extract the sources, in case the URL does not identify it.

      - `commit`: mandatory string for VCS origins that describes the
        VCS-specific revision to be checked out (a git/hg hash, a svn
        revision).

   Examples of origin tables:

   ```toml
   # Clone a git repository
   [origin]
   url = "git+https://github.com/example-user/example-project"
   commit = "ec8b267bb8b777c6887059059924d823e9443439"
   ```

   ```toml
   # Download and extract a source archive
   origin = "https://example.org/0123456789"
   archive-name = "archive.tar.gz"
   hashes = ["sha512:bf6082573dc537836ea8506a2c9a75dc7837440c35c5b02a52add52e38293640d99e90a9706690591f8899b8b4935824b195f230b3aa1c4da10911e3caf954c04ac"]
   ```

 - `available`: optional dynamic boolean expression.  If it evaluates to
   `false`, the package is not available for the current platform.

 - `notes`: optional string. Provides miscellaneous information about this
   release. For instance:

   ```json
   notes = "Experimental version"
   ```

 - `configuration` optional table to control crate configuration code
   generators:

   For more information on crate configuration, see [Using crate
   configuration](#using-crate-configuration).

      - `disabled`: Completely disable configuration code generation for the
        crate (default: `false`)

      - `output_dir`: Path to the directory where the configuration code will
        be generated, relative to the crate root (default: `config`).

      - `generate_ada`: Enable generation of Ada configuration (default:
        `true`).

      - `generate_gpr`: Enable generation of GPR file configuration (default:
        `true`).

      - `generate_c`: Enable generation of C configuration (default: `true`).

      - `auto_gpr_with`: Enabled generation of list of withed project in the
        GPR file configuration (default: `true`).


 - `configuration.variables` optional table of crate configuration variable
   definitions.

   For more information on crate configuration, see [Using crate
   configuration](#using-crate-configuration).

   The keys of the table are names of the variables. Variable definitions
   themselves are tables with the following entries:

      - `type`: mandatory string which defines the type of the variable, it can
        be:

         - `String`: any string

         - `Boolean`: either `True` or `False`

         - `Enum`: enumeration type

         - `Integer`: an integer value that can be encoded in 64-bit

         - `Real`: a real value that can be encoded in IEEE 754 binary64

      - `default`: optional default value for the variable. Will be used if no
        crates explicitly set a value for this variable. Must be a valid value
        for the type.

      - `first`: (optional) for `Real` and `Integer` types only. Defines the
         lower bound of valid values for the type (inclusive).

      - `last`: (optional) for `Real` and `Integer` types only. Defines the
         upper bound of valid values for the type (inclusive).

      - `values`: mandatory for `Enum` types. An array of strings containing
        all the possible values for the enumeration.


   Example:
   ```toml
   [configuration.variables]
   Device_Name = {type = "String", default = "no device name"}
   Print_Debug = {type = "Boolean", default = false}
   Debug_Level = {type = "Enum", values = ["Info", "Debug", "Warn", "Error"], default = "Warn"}
   Buffer_Size = {type = "Integer", first = 0, last = 1024, default = 256}
   Max_Power   = {type = "Real", first = 0.0, last = 100.0, default = 50.0}
   ```
 - `configuration.values` optional table of variables assignment:

   The keys of the table are crate names, and entries are sub-tables of
   `variable_name` and `value`. The type of the value has to match the
   definition of the variable type.

   Example:
   ```toml
   [configuration.values]
   crate_1.var1 = 42
   crate_1.var2 = true
   crate_2.var1 = "Debug"
   ```

## External releases

The above information applies to regular releases distributed from sources
(that is, the Ada projects whose distribution is the main Alire goal). Some
special supporting releases also exist that are described differently.

A release is considered "external" when it is not built from sources and,
furthermore, its semantic version cannot be known until run time. Hence, the
availability and version of these releases is detected by `alr`.

Several definitions for these external releases may exist so they are
defined in a manifest as a vector with key `external`:

```toml
[[external]]
# Common entries to all externals
kind = "hint" # One of several predefined external kinds
hint = "Please install SDL in your platform from source or system packages"
# Specific external kind parameters might follow
```

All external kinds can define these regular properties:

 - `available`: when defined, it restricts the external detection to the given
   environment conditions.

 - `hint`: optional dynamic string containing an explanation for the user on
   how to make the external entity available. This explanation is shown on request
   with `alr show --external`, or after `alr get`, for any external dependency
   that could not be detected.

### External kinds: hints

A plain undetectable external kind intended to simply serve as a hint. For
crates that are known to be unavailable through Alire, it serves to
provide a generic or customized hint to the user. It has no specific
fields, other than the common ones just described. Its key is `"hint"`:

```toml
[[external]]
kind = "hint" # Identifies this external kind
# Bare minimum external. Optionally, the hint/available fields can be used.
```

### External kinds: command-line tools

This external kind is used to describe commands that can be run in the system,
and that are able to provide their own version via some particular invocation.
Their specific fields are (all mandatory):

```toml
kind = "version-output" # Identifies this external kind

version-command = ["gnat", "--version"]
# Invocation that will provide the version when the tool is available

version-regexp  = "^GNAT ([\\d\\.]+).*|^GNAT Community ([\\d]{4}).*"
# TOML-escaped GNAT.Regpat-compatible regular expression. Parenthesized
# matches will cause the matched expression to be parsed as the Semantic
# Version of the tool.
```

### External kinds: system packages

Systems that have their own package manager (e.g. Linux) can readily provide
many complex dependencies still unpackaged as source code in Alire. Alire can
use these on supported platforms (at this time, Debian & Ubuntu. Do not
hesitate to contact us if you would like to maintain other distributions)
during resolution.

A system external gives a list of platform package names that supply the
dependency natively. The platform package manager will be used to detect their
availability and version. To that effect, the `origin` field is used (which can
accept dynamic expressions in this context):

```toml
kind = "system" # Identifies this external kind
origin = ["libncursesada3", "libncursesada5"]
# As versions appear this list will grow. To speed up detection, dynamic
# expressions may become recommended for certain system packages.
```

For Ada pre-compiled system libraries that require the platform compiler for
linking (e.g., in Debian/Ubuntu), and that cannot be used with other GNAT
compilers, this should be expressed with the `available` property, e.g.:

```toml
available.'case(toolchain)'.user = false
# `available` defaults to true, so it is enough to flag the user toolchains
```

## Parameters

 - `os`: name of the OS. Currently supported values are: `linux`, `macos` and
   `windows`.

 - `distribution`: name of the Linux distribution, or `none` if running on a
   different OS. Currently supported values are: `debian`, `ubuntu`.

 - `toolchain`: takes `system` value in distributions with the system Ada
   compiler first in PATH (GNAT FSF in Debian/Ubuntu), `user` otherwise (GNAT
   Community editions, other cross-target toolchains).

 - `word-size`: architecture word size. Currently supported values are:
   `bits-32`, `bits-64`, `bits-unknown`

## Using crate configuration

`Alire` provides a mechanism for crates to expose a list of variables that can
be set by other crates depending on them. The configuration variables will then
be converted to Ada, C and GPR source files that can be used to change the
behavior or feature set of the code.

Let's start with a simple example. A crate named `test` can print debug log on
the console. However printing on the console has a performance impact, for an
embedded project it can even have a significant code size impact. Therefore it
would be best if this logging can be disabled/enabled at compile time.

To achieve this, a crate maintainer can define a configuration variable in the
crate manifest `alire.toml`. The definition will be like so:
```toml
[configuration.variables]
Enable_Logs = {type = "Boolean", default = false}
```
A single variable of type `Boolean` with a default value of `false`.

From this definition, `Alire` will generate various source files, including an
Ada package specification:

```ada
package Test_Config is
   Enable_Logs : constant Boolean := False;
end Test_Config;
```

In the crate source code, this configuration package can be used like so:
```ada
   if Test_Config.Enable_Logs then
      Print_Log ("This is a log message.");
   end if;
```

If one of the crates depending on `test` sets the configuration variable to
`true`, e.g.:

```toml
[configuration.values]
test.Enable_Logs = true
```

The constant value will change in the generated configuration package:
```ada
package Test_Config is
   Enable_Logs : constant Boolean := True;
end Test_Config;
```
Which will enable logging in the `test` crate.

It is possible for multiple depending crates to set `test.Enable_Logs` to the
same value, however if two depending crates set the variable to a different
value then the configuration is invalid and `Alire` will print an error. If no
depending crates set the `test.Enable_Logs` variable, then its default value is
used.

### When to use crate configuration?

Usually when something has to be static or known at compiler-time, either for
performance or memory usage.

### When _not_ to use crate configuration?

When the Ada languages provides a better alternative. There are many ways to
provide an Ada API that will result in compile time optimization or static
memory usage.

For instance, discriminants are an effective way to let the user define the
size of a buffer:

```ada
   type Buffered_Thing (Size : Positive) is private;
private
   type Buffer_Array is array (Positive range <>) of Unsigned_8;
   type Buffered_Thing (Size : Positive) is record
      Buf : Buffer_Array (1 .. Size);
   end record;
```

With this definition, users are then able to allocate either statically, on the
stack or on the heap depending on their project.

```ada
   Thing : Buffered_Thing (Size => 256);
```

### Use cases

#### Log levels

Enumerations variables in crate configuration can be used to set a level of log
verbosity:
```toml
[configuration.variables]
Log_Level = {type = "Enum", values = ["Info", "Debug", "Warn", "Error"], default = "Warn"}
```

#### Buffer size

Integer variables can be used the define the size of a static buffer:

```toml
[configuration.variables]
Buffer_Size = {type = "Integer", first = 0, last = 1024, default = 256}
```
This is useful in particular for embedded projects where compile time memory
usage is preferred over dynamic allocation.

#### Server URL

String variables can be used to define the URL of a website or service:

```toml
[configuration.variables]
URL_Name = {type = "String", default = "example.com"}
```

#### PID coefficients 

Real variables can be used for PID coefficients:
```toml
[configuration.variables]
Proportional = {type = "Real"}
Integral = {type = "Real"}
Derivative = {type = "Real"}
```
#### Worst case allocation

Integer variable can be used to define The maximum length of file names in a
file-system:
```toml
[configuration.variables]
Max_Filename_Length = {type = "Integer", first = 5, last = 128}
```

#### Select algorithm in GPR project file

Crate configuration also generates a GPR project file, therefore it can be used
to control which units are compiled in the project.
```toml
[configuration.variables]
Sort_Algorithm = {type = "Enum", values = ["bubble", "quick", "merge"]}
```

The generated GPR will look something like this:
```ada
project Test_Config is
   type Sort_Algorith_Kind is ("bubble", "quick", "merge");
   Sort_Algorith : Debug_Level_Kind := "quick";
end Test_Config;
```

It can be used in the main GPR file like so:

```ada
   package Naming is
      for Body ("Test.Sort") use "test-sort__" & Test_Config.Sort_Algorith;
   end Naming;
```
With the files `test-sort__bubble.adb`, `test-sort__quick.adb` and
`test-sort__merge.adb` each implementing a different algorithm.


## Further reading ##

You can inspect [index files](https://github.com/alire-project/alire-index) to
get an idea of how projects are included into the catalog.
