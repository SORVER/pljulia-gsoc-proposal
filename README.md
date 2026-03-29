
# GSoC'26 PostgreSQL Proposal
## PL/Julia: Adding Features to the Julia Procedural Language

![GSoC Logo](https://upload.wikimedia.org/wikipedia/commons/0/08/GSoC_logo.svg)



**Organization:** PostgreSQL

**Project:** PL/Julia, Julia Procedural Language for PostgreSQL

**Name:** Abdelsalam Motsafa

**Email:** abdelsalam.contact@gmail.com

**GitHub:** SORVER

**Timezone:** Cairo, Egypt, UTC/GMT +2 hours

---

## Abstract

PL/Julia is a PostgreSQL extension that allows users to write functions using the Julia programming language. As PostgreSQL supports many procedural languages, PL/Julia is still under development and needs improvements to support newer Julia versions and become stable like other extensions listed in the PL Matrix.

---

## Introduction to PL/Julia

An open-source project that lets you use Julia inside PostgreSQL. It was created by the PostgreSQL community and was developed more as a past Google Summer of Code project.

The idea is simple: Julia is great for heavy calculations and data work, but normally you'd have to leave the database to use it. PL/Julia lets you do all that inside PostgreSQL, so your data and your code live in the same place.

When we write SQL code, PostgreSQL forwards every Julia function call to its procedural language handler, which then runs the corresponding Julia code through the Julia runtime, as you see in the diagram below.

![PL/Julia call flow](https://i.ibb.co/N2fvm6HG/Untitled-2025-02-14-2301-1.png)

PostgreSQL procedural languages are built around three components: a call handler that executes your function when called, a validator that catches obvious errors at definition time, and an inline handler that lets you run anonymous code blocks on the fly via `DO...LANGUAGE`.

![PL/Julia usage options](https://i.ibb.co/v0D39sC/Postgre-SQL-Function-2026-03-22-081049.png)

With PL/Julia, you can:
- Run Julia functions directly in SQL.
- Make reusable Julia functions in the database.
- Play with Julia code on the fly without defining a function.

It's a simple idea, but it opens the door to doing scientific computing, data analysis, and heavy calculations all inside the database, without switching tools.
PL/Julia will be a great addition because of Julia's strengths in high-performance computing and scientific computing.

---

## Current State of PL/Julia

PL/Julia builds successfully against PostgreSQL 13 and Julia 1.6. While these versions are older, the extension already has the core features needed for a production-ready procedural language.

### 1. Type Mapping

The extension handles the complex task of translating data between PostgreSQL's C-based types and Julia's high-level objects:
- **Primitives:** Direct mapping for integers, floats, booleans, and text.
- **Arrays:** Native support for multi-dimensional PostgreSQL arrays, converted to Julia arrays while respecting Julia's column-major memory layout.
- **Composite Types:** Database rows and custom types are passed into Julia as Dictionaries, so you can access columns by name.
- **Null Handling:** SQL `NULL` values are correctly mapped to Julia's `nothing`.

| PostgreSQL | Julia |
|---|---|
| `NULL` | `nothing` |
| `boolean` | `Bool` |
| `int` | `Int64` |
| `numeric` | `BigFloat` |
| `text`, `varchar` | `String` |
| other scalar type | `String` |
| Composite type (in) | `Dictionary` |
| Composite type (out) | `Tuple` |
| Array (row-major) | Array (column-major) |

### 2. Execution Modes

- **Set-Returning Functions (SRF):** Users can write functions that return multiple rows by using the `return_next()` function within Julia code.
- **Anonymous Blocks (DO Statements):** Supports the `DO $$...$$ LANGUAGE pljulia;` syntax, so users can run one-off Julia scripts without defining a permanent function.
- **Trigger Support:** Supports row-level, statement-level, and event triggers. Julia functions can catch data changes and use the `TD_NEW` and `TD_OLD` dictionaries to modify data on the fly.

### 3. Database Integration (SPI)

PL/Julia exposes the Server Programming Interface (SPI) to the Julia environment, allowing Julia functions to "call back" into the database:
- `spi_exec`: Run any SQL query from Julia.
- `spi_prepare` & `spi_exec_prepared`: Support for prepared statements to improve performance and security.
- Cursor-related functions for going through large result sets.

### 4. Global State Persistence

One of PL/Julia's most powerful features is the Global Dictionary (`GD`).
This is a dictionary that stays alive as long as the database connection.

**Use Case:** You can load a heavy Machine Learning model or a large lookup table into `GD` once, and reuse it across thousands of function calls without loading it again every time.

---

## Problem Statement

PL/Julia still targets older Julia versions and needs work before it can be considered production-ready. I recently contributed Julia 1.12 support with PostgreSQL 14/15, and that has been merged. The sections below describe the specific issues I found while working with the codebase.

PL/Julia still targets Julia 1.6 and PostgreSQL 13. The CI pipeline on GitLab and the Dockerfile were built around those versions, and anything newer is either untested or breaks. I recently contributed support for Julia 1.12 with PostgreSQL 14 and 15, and that work has been merged. When I tested against PostgreSQL 16, I found that 5 of 24 regression tests fail because Julia's `-fvisibility=hidden` flag hides SPI callback symbols from the shared library ([Issue #33](https://github.com/pljulia/pljulia/issues/33)). Compatibility across all supported versions is still an open problem that should be fixed in this project.

A proper testing matrix covering different Julia and PostgreSQL versions would help ensure stability across environments. This will help a lot for anyone who wants to contribute to PL/Julia.

The main reason is that Julia has no stable C API (see [Julia issue #56805](https://github.com/JuliaLang/julia/issues/56805)). Every major release can introduce breaking changes, and they do. In 1.11, `jl_arrayref()` and `jl_arrayset()` were removed and replaced with functions that take arguments in a different order. In the upcoming 1.13.0-beta2, `jl_function_t` has been dropped from the public headers entirely, so any code that uses it won't compile. The fix is simple (replace `jl_function_t*` with `jl_value_t*`, since functions are just values in Julia), but it shows how fast things move. Without a CI matrix covering multiple versions, these breakages only show up when someone happens to try building against a newer release.

There's no CI/CD pipeline on the GitHub repository. Build failures and regressions against new Julia or PostgreSQL releases go unnoticed until someone tries a manual build. The original GitLab CI only covered Julia 1.6 and 1.7 with PostgreSQL 12 and 13.

There are also stability problems in the C bridge code. Julia has a garbage collector, and when you hold Julia values from C, you have to tell Julia not to free them using `JL_GC_PUSH`/`JL_GC_POP` macros. I looked through `pljulia.c` and found about 44 `jl_value_t*` variables, but only 1 is actually protected. The rest can be freed at any time while the code is still using them. Under normal conditions this works by luck, but under memory pressure it causes random crashes that are very hard to track down.

On top of that, Julia never gets a chance to clean up when PostgreSQL shuts down a backend process. Julia needs to release memory, close files, and reset signal handlers through a function called `jl_atexit_hook(0)`. But in PL/Julia's code, this call is commented out (line 1234), and no cleanup is registered through PostgreSQL's `on_proc_exit()` system either. So Julia just stops without cleaning up, which can leak memory and leave things in a bad state.

Error reporting is another issue. Every error goes through `elog(ERROR, message)` with no SQLSTATE codes, no `DETAIL`, no `HINT`. Production PostgreSQL extensions usually use `ereport()`, which gives tools like pgAdmin and monitoring systems enough information to tell different failures apart. Also, the way Julia exceptions get formatted (by calling `jl_eval_string("sprint(showerror, ...)")`) can itself fail silently, and the full Julia stack trace is always lost. Debugging a failed function in production is basically guesswork.

Type handling has problems too. The code checks if a Julia value is a dictionary by comparing the type name as a string: `strcmp(jl_typeof_str(ret), "Dict") == 0`. This works for `Dict{Any, Any}`, but fails quietly for typed dictionaries like `Dict{String, Int64}`, because the string includes the type parameters and the comparison fails. On the numeric side, `smallint` and `integer` are both boxed as Julia `Int64` instead of `Int32`. There is a comment in `convert_args.c` that says "should be jl_box_int32 but this will be taken care of along with the rest of the input conversion. For a very little while let's leave it like this." That was never done.

Several important PostgreSQL types aren't supported at all: `DATE`, `TIMESTAMP`, `TIMESTAMPTZ`, `TIME`, `TIMETZ`, `INTERVAL`, `bytea`, `UUID`, and `JSONB`. If you pass any of these into a Julia function, they show up as raw strings. Array parameters of these types don't work either.

Transaction control is missing. There's no `spi_commit()` or `spi_rollback()`, so Julia code has no way to manage transactions. PL/Python, PL/Perl, and PL/Tcl all provide that. Also, there's a misleading comment in `pljulia_spi_query` that reads "Commit the inner transaction, return to outer xact context," placed next to a call to `SPI_finish()`. But `SPI_finish()` doesn't commit anything, it just closes the SPI connection. This looks like a copy-paste from PL/Tcl's code.

Package loading at startup is also fragile. When PL/Julia initializes, it runs `using Pkg; collect(p.name for p in values(Pkg.dependencies()) if p.is_direct_dep)` to find installed packages, then loops through them and calls `jl_eval_string("using PackageName")` for each one. This has several problems: it runs `using Pkg` on every PostgreSQL backend startup (which is slow), it builds the `using` command by string concatenation with `strcpy`/`strcat` (which could overflow for long package names, though a recent fix capped the buffer), and if any single package fails to load, the error is ignored and the rest of the packages still load. There's no way for the user to control which packages get loaded or to skip package loading entirely.

Some features that other procedural languages have are also missing. PL/Python provides both a global dictionary (`GD`) and a per-function private dictionary (`SD`) for caching state across calls, but PL/Julia only has `GD`. SQL quoting helpers like `quote_literal()` and `quote_ident()` don't exist, which pushes users toward string concatenation with the injection risks that come with it. And `spi_exec_prepared` always requires a row limit. There's no way to get a cursor back for going through large result sets.

PL/Julia is not yet at the level of stability that PL/Python or PL/Java have reached. The goal of this project is to close that gap and make PL/Julia a reliable extension for everyone, In Sha Allah.

---

## Proposed Solution

My plan, In Sha Allah, is to make PL/Julia work with current versions of Julia and PostgreSQL, add the missing data types, and bring it closer to what other procedural languages like PL/Python already offer. I already submitted a pull request that makes PL/Julia build with Julia 1.12 and PostgreSQL 14/15, so I have spent time inside the codebase and I know where the problems are.

The work has four parts, sorted by what matters most:

1. **Fix stability and compatibility:** make the existing tests pass on new versions
2. **Add missing data types:** dates, timestamps, binary data, etc.
3. **Add missing features:** transaction control, better errors, CI/CD
4. **Stretch goals:** trusted language variant, per-function storage, subtransactions

---

## Expected Challenges

### Garbage Collection Safety

Julia has a garbage collector (GC). When we use Julia values from C code, we have to tell Julia "I am still using this value, do not free it." We do this with `JL_GC_PUSH` and `JL_GC_POP` macros.

I looked through `pljulia.c` and found that there are about 44 places where the code holds a Julia value (`jl_value_t*`), but only 1 of them is actually protected with `JL_GC_PUSH`. The rest are not protected at all. For example, in the `pljulia_spi_exec` function, the code creates a result array and then fills it inside a loop. Any of the Julia calls inside that loop could trigger the GC, which could free the result array while we are still writing to it. This leads to crashes that are hard to reproduce.

The fix is to go through every function and add the missing `JL_GC_PUSH`/`JL_GC_POP` pairs, In Sha Allah. Julia gives us `JL_GC_PUSH1` up to `JL_GC_PUSH6` for protecting up to six values, and `JL_GC_PUSHARGS` for bigger cases. To test that the fix works, I will, In Sha Allah, write stress tests that allocate a lot of values in a loop to make the GC run more often. I can also build a debug version of Julia (with `WITH_GC_DEBUG_ENV=1` flag) that forces GC on every allocation, which makes these bugs show up right away.

---

### Julia Version Compatibility

Julia changed its C API in version 1.11 and this broke PL/Julia. The main changes:

- The functions `jl_arrayref()` and `jl_arrayset()` were removed completely. The new functions are called `jl_array_ptr_ref()` and `jl_array_ptr_set()`. The problem is that the argument order in `jl_array_ptr_set` is different, the value and index are swapped. If you just rename the function without fixing the order, the code compiles fine but breaks the data without any warning.
- The internal memory layout of arrays changed in 1.11. Any code that accesses array memory directly (pointer arithmetic) is not safe anymore.
- Julia does not have an official "stable C API." There is an open issue about this ([Julia #56805](https://github.com/JuliaLang/julia/issues/56805)). This means future Julia versions could break things again, which is why we need CI testing across multiple versions.

My PR already handles the function renames. Julia 1.12 (released October 2025) did not introduce new breaking changes to the embedding API, so the 1.11 fixes work for 1.12 too. What remains is to check all code paths for old array assumptions and set up CI to test automatically.

---

### Dict Type Detection

The code checks if a Julia value is a dictionary by comparing the type name as a string:

```c
#define jl_is_dict(ret) (strcmp(jl_typeof_str(ret), "Dict") == 0)
```

This works for `Dict{Any, Any}`, but if a user returns `Dict{String, Int64}`, the type name becomes `"Dict{String, Int64}"` and the comparison fails. The code then does not know how to handle the value.

The fix is to use Julia's own type checking function `jl_isa()`, which handles subtypes correctly:

```c
jl_value_t *abstract_dict_type = jl_eval_string("AbstractDict");
jl_isa(ret, abstract_dict_type)  // works for all Dict types
```

---

### Date and Time Types

PostgreSQL has six date/time types (`DATE`, `TIMESTAMP`, `TIMESTAMPTZ`, `TIME`, `TIMETZ`, `INTERVAL`) but PL/Julia does not support any of them. Right now, if you pass a date to a Julia function, it arrives as a raw string that you have to parse yourself. Julia has matching types in its `Dates` module, so the mapping is natural:

- **PostgreSQL to Julia:** Get the string form of the value using PostgreSQL's output function, then parse it in Julia with `Dates.Date(str)`, `Dates.DateTime(str)`, etc.
- **Julia to PostgreSQL:** Convert the Julia date to a string, then pass it through PostgreSQL's input function.

One thing to be careful about is `TIMESTAMPTZ`. PostgreSQL stores these as UTC internally but shows them in the session timezone. Julia's `DateTime` does not know about timezones. The safest approach is to always pass UTC to Julia and document this clearly. This is what PL/Python does too.

For `INTERVAL`, Julia has no single interval type. It uses things like `Dates.Day`, `Dates.Hour`, etc. The plan is to pass intervals as a Julia dictionary with keys like `"days"`, `"hours"`, `"seconds"`. This follows the same pattern PL/Julia already uses for composite types.

---

### Arrays as IN Parameters

PL/Julia can return arrays, but accepting arrays of the new types as input parameters needs extra work. The existing code converts array elements one by one through a function called `pg_oid_to_jl_value()` in `convert_args.c`. Right now this function only knows about integers, floats, booleans, and text. If you pass an array of dates, each element falls through to the default case and arrives as a plain string instead of a `Dates.Date`.

The good news is that fixing `pg_oid_to_jl_value()` for scalars automatically fixes arrays too, because arrays use the same function for each element. So adding `DATE` support for scalar parameters means `DATE[]` arrays also work without extra code.

After this work, things like this should just work:

```sql
CREATE FUNCTION first_date(dates DATE[]) RETURNS DATE AS $$
    minimum(dates)
$$ LANGUAGE pljulia;
```

I will, In Sha Allah, write tests for each new type both as a scalar and as an array.

---

### Adding `bytea` Support

`bytea` is how PostgreSQL stores binary data. The natural Julia type for this is `Vector{UInt8}`.

PostgreSQL stores `bytea` as a "varlena", which is a structure with a size header followed by the raw bytes. On the C side, we use `VARDATA` and `VARSIZE_ANY_EXHDR` to get the bytes and their length, then copy them into a Julia `UInt8` array using `jl_ptr_to_array_1d`.

Going back from Julia to PostgreSQL, we read the bytes from the Julia array using `jl_array_data`, allocate a PostgreSQL varlena with `palloc`, copy the bytes in, set the size with `SET_VARSIZE`, and return it.

The important thing here is memory ownership. The Julia array owns its memory buffer, and PostgreSQL has its own memory system. We have to copy the data between them, not share pointers, because Julia's GC could free the buffer at any time.

---

### Transaction Control

Other procedural languages like PL/Python, PL/Perl, and PL/Tcl let users commit or rollback transactions from inside their code. PL/Julia does not have this, and it is listed as a limitation in the README.

PostgreSQL has `SPI_commit()` and `SPI_rollback()` functions available since version 11. I will, In Sha Allah, write C wrapper functions and register them so Julia code can call them:

```c
jl_eval_string("spi_commit() = ccall(:pljulia_spi_commit, Cvoid, ())");
jl_eval_string("spi_rollback() = ccall(:pljulia_spi_rollback, Cvoid, ())");
```

There are two things to be careful about. First, PostgreSQL only allows commit and rollback inside **procedures** (`CREATE PROCEDURE`) and `DO` blocks, not inside regular functions. This is a PostgreSQL rule, not something we can change. The code will need to check the calling context and give a clear error message if someone tries to use it in the wrong place.

Second, after a commit, the database connection resets. Any prepared plans or open cursors from before the commit stop working. We need to handle this, either by clearing the cached plans or by documenting that users need to re-prepare their plans after a commit. PL/Perl does it the same way, so we can follow their approach.

---

### Package Loading at Startup

When PL/Julia starts up in `_PG_init()`, it loads all installed Julia packages by running:

```c
jl_value_t *packages = jl_eval_string("using Pkg; collect(keys(Pkg.installed()))");
```

But `Pkg.installed()` was deprecated in Julia 1.4, so every time a PostgreSQL backend starts, it prints:

```
Warning: Pkg.installed() is deprecated
```

I opened [Issue #22](https://github.com/pljulia/pljulia/issues/22) about this. The fix is to replace it with `Pkg.dependencies()`, which is the current supported API and does not give a warning. There are two options:

- **Option A (no filter):** `collect(p.name for p in values(Pkg.dependencies()))` loads all packages including transitive dependencies. For example, if you add `DataFrames`, it will also load `Tables` and other packages that `DataFrames` depends on. This is slower.
- **Option B (with filter):** `collect(p.name for p in values(Pkg.dependencies()) if p.is_direct_dep)` loads only packages the user explicitly added with `Pkg.add()`. This is faster and makes more sense.

This fix is not merged yet. Then it loops through the result and calls `jl_eval_string("using PackageName")` for each package. On top of the deprecated API, there are more problems:

- **It's slow.** `using Pkg` itself takes time, and it runs on every new PostgreSQL backend connection. For a server handling many connections, this adds up.
- **No error handling.** If a package fails to load (maybe it was removed or is broken), the error is not reported to the user. The loop just continues.
- **No user control.** There's no GUC (PostgreSQL configuration variable) to let users choose which packages to load, or to disable package loading entirely for faster startup.
- **String concatenation for `using` commands.** The code builds the command with `strcpy`/`strcat` into a fixed buffer. A recent fix capped the buffer size, but this is still not great. The same unsafe `strcpy`/`strcat` pattern also appears in `pljulia_compile` (function compilation), the trigger path, and the event trigger path. All of these should be replaced with PostgreSQL's `StringInfo` API.

The fix is to first replace `Pkg.installed()` with `Pkg.dependencies()` (Option B), and then add a PostgreSQL GUC like `pljulia.preload_packages` that accepts a comma-separated list of package names (or `'*'` for all, `''` for none). This gives users control over startup time and avoids loading packages they don't need. Error handling should also be added so that if a package fails to load, the user sees a `WARNING` with the package name and the Julia error message, instead of silent failure.

---

### Error Reporting

When Julia code throws an error, PL/Julia catches it and shows it as a PostgreSQL error. But the way it formats the error message is not reliable:

```c
jl_eval_string("sprint(showerror, ccall(:jl_exception_occurred, Any, ()))")
```

The problem is that `jl_eval_string` itself can throw an error, which means the error formatting can fail without any warning. Also, the Julia stack trace is lost completely, so if a function crashes in production there is almost no information to debug it.

The fix is to use Julia's C API functions directly instead of going through `eval_string`. We call `jl_current_exception()` to get the error, then use `jl_call2` to call Julia's `sprint` and `showerror` functions. For the stack trace, we call `Base.catch_backtrace()` and put it in PostgreSQL's error `DETAIL` field so it shows up in the logs.

---

## Milestones

### Milestone 1: Stability and Compatibility

This is the most important milestone. Everything else depends on it.

- Audit all of `pljulia.c` for GC safety and add missing `JL_GC_PUSH`/`JL_GC_POP` (only 1 out of ~44 Julia values is currently protected)
- Fix Dict and BigFloat type detection to use `jl_isa` instead of string comparison
- Fix `smallint` and `integer` boxing to use `jl_box_int32` instead of `jl_box_int64` (there is a TODO comment in `convert_args.c` that was never finished)
- Register `jl_atexit_hook(0)` through `on_proc_exit()` so Julia shuts down properly when a database backend exits
- Add tracking for SPI connection depth to prevent "SPI already connected" errors
- Fix package loading: add a GUC for controlling which packages load at startup, add error handling for failed package loads
- Make sure all 26 existing tests pass on Julia 1.9, 1.10, 1.11, 1.12 with PostgreSQL 14, 15, 16, 17
- Set up GitHub Actions CI with this version matrix

**Goal:** All 26 tests pass on all version combinations. CI runs on every pull request.

---

### Milestone 2: Missing Data Types

Once the base is stable, add the types people actually need.

- `DATE` → `Dates.Date`
- `TIMESTAMP` → `Dates.DateTime`
- `TIMESTAMPTZ` → `Dates.DateTime` (UTC)
- `TIME` / `TIMETZ` → `Dates.Time`
- `INTERVAL` → `Dict` with named fields
- `bytea` → `Vector{UInt8}`
- `UUID` → `String`
- `JSONB` / `JSON` → `String` (users parse it with any Julia JSON package they want)
- `NUMERIC` Infinity / NaN → Julia `Inf`, `-Inf`, `NaN`
- Arrays as IN parameters for all new types (this comes for free because arrays convert elements through the same function we are already fixing)

For each type: write regression tests for the normal case, NULL values, and edge cases (midnight for time, leap day for date, `Inf` for numeric).

**Goal:** All listed types work in both directions. Each type has regression tests.

---

### Milestone 3: Missing Features

Add features that other procedural languages already have.

- **Transaction control:** `spi_commit()` and `spi_rollback()` (only in procedures and DO blocks)
- **Better error messages:** Full Julia stack traces in PostgreSQL's error DETAIL field, switch from `elog()` to `ereport()` with proper SQLSTATE codes
- **Cursor from prepared plans:** `spi_exec_prepared(plan, args)` without a limit, returning a cursor
- **`SD` dictionary:** Per-function private storage, like PL/Python has. Right now PL/Julia only has `GD` which is shared across all functions. `SD` would let each function keep its own data separately
- **SQL quoting helpers:** `quote_literal()`, `quote_ident()`, `quote_nullable()` so users can build dynamic queries safely
- **Extension upgrade scripts:** `pljulia--0.8--0.9.sql` so `ALTER EXTENSION pljulia UPDATE` works without dropping everything
- **Error path tests:** Tests that check what happens when things go wrong (wrong types, NULL in unexpected places, bad SQL, Julia exceptions)
- **Test coverage gaps I found:** NULL input is completely untested (SQL `NULL` maps to Julia `nothing` correctly, but no test passes NULL as input). Trigger `SKIP` return value is documented in the README but has no test. `STRICT` functions (NULL in → NULL out without running Julia code) are a standard PostgreSQL feature with zero coverage in the test suite

**Goal:** Transaction control works. Errors show stack traces. The extension can be upgraded.

---

### Milestone 4 (Stretch): Trusted Variant and Subtransactions

These are stretch goals, only if there is time after the first three milestones.

- **Trusted variant (`pljulia`):** Right now, PL/Julia is "untrusted", which means Julia code can read files, run shell commands, and access the network. Only superusers can create PL/Julia functions. A trusted version would block these dangerous operations so that regular database users can create functions safely. PL/Perl already does this (it has trusted `plperl` and untrusted `plperlu`), but PL/Python never managed to do it. JuliaC.jl, a new tool for compiling Julia into standalone native libraries, may open a path here. Its `--compile-ccallable` and `--trim` options could help define clear safety boundaries. A small proof-of-concept would be enough to validate the approach during this project, with full implementation as future work.
- **Subtransactions:** Right now, if Julia code fails partway through, the whole transaction fails. Subtransactions let you wrap a risky piece of code in a savepoint, so if it fails you can roll back just that part and keep going. This uses PostgreSQL's savepoint system under the hood.
- **More types:** `ENUM` (mapped to Julia strings), domain types (custom types built on top of existing ones), and `RECORD` as a return type from set-returning functions.

**Goal:** Trusted variant proof-of-concept. Subtransactions work.

---

## Timeline

> Standard GSoC timeline (12 weeks).

### Community Bonding (1 May – 24 May)

- Get in touch with the community and discuss the approach with mentors
- Read through all of `pljulia.c` and `convert_args.c` carefully, write down every issue I find
- Set up my development environment: PostgreSQL 14, 15, 16, 17 from source; Julia 1.9, 1.10, 1.11, 1.12
- Run the 26 tests on each combination and record what passes and what fails
- Talk with mentors about which parts of Milestone 3 are most important if time gets tight

---

### Milestone 1: Weeks 1–4 (25 May – 21 June)

**Week 1–2:** GC safety audit
- Go through every function that holds a Julia value across a call (~44 unprotected variables)
- Add all missing `JL_GC_PUSH`/`JL_GC_POP`
- Write stress tests that allocate a lot to make GC run during dangerous moments
- Optionally build a debug Julia for deeper verification

**Week 3:** Julia API fixes, type detection, shutdown, and SPI
- Make sure array API migration is complete and correct everywhere
- Replace `strcmp`-based Dict detection with `jl_isa(val, AbstractDict)`
- Fix BigFloat detection the same way
- Register `jl_atexit_hook` through `on_proc_exit()`
- Add SPI connection depth tracking

**Week 4:** Package loading, CI, and compatibility testing
- Fix package loading: add `pljulia.preload_packages` GUC, add error handling for failed loads
- Set up GitHub Actions with the Julia x PostgreSQL version matrix
- Run all tests on all combinations, fix whatever breaks

---

### Milestone 2: Weeks 5–8 (22 June – 19 July)

**Week 5–6:** Date, timestamp, time, and interval types
- Add `DATE`, `TIMESTAMP`, `TIMESTAMPTZ`, `TIME`, `TIMETZ`, `INTERVAL`
- Write tests for each, including edge cases (leap days, DST, midnight, NULL)

**Week 7:** Binary data, UUID, JSON, and numeric edge cases
- Add `bytea` to `Vector{UInt8}` conversion (both ways)
- Add `UUID` as string passthrough
- Add `JSONB` / `JSON` as string passthrough
- Fix `NUMERIC` to handle Infinity and NaN
- Write tests

**Week 8:** Array parameters and review
- Make sure all new types work as array elements too
- Test multi-dimensional arrays with new types
- Fix issues found during mentor review

> **Note:** Midterm evaluation is due July 10. By this point, In Sha Allah, all stability fixes will be done and most types will be added.

---

### Milestone 3: Weeks 9–12 (20 July – 16 August)

**Week 9–10:** Transaction control and error reporting
- Write `pljulia_spi_commit()` and `pljulia_spi_rollback()`
- Register them in `_PG_init()`
- Add check that they are only called from procedures and DO blocks
- Handle plan cache invalidation after commit
- Replace `eval_string(sprint(...))` with direct C API calls
- Switch from `elog()` to `ereport()` with SQLSTATE codes and DETAIL
- Capture Julia backtrace and put it in the error DETAIL

**Week 11:** SD dictionary, SQL quoting, and upgrade scripts
- Add per-function `SD` dictionary
- Add `quote_literal`, `quote_ident`, `quote_nullable`
- Write `pljulia--0.8--0.9.sql` migration script

**Week 12:** Error tests, documentation, and final cleanup
- Add tests for wrong types, bad SQL, exceptions, unexpected NULLs
- Update README with all new types and features
- Write final project report: what was done, what still needs work, what I recommend for the next contributor

> **Final submission deadline:** August 17–24, 2026

---

### Stretch Goals (if time allows)

> These will only be done if all issues above are fixed, In Sha Allah.

- Trusted variant proof-of-concept using JuliaC.jl
- Subtransaction support
- More types (`ENUM`, domain types, `RECORD`) based on mentor feedback
- Cursor from prepared plans: `spi_exec_prepared(plan, args)` without a limit

---

**Note:** Milestones 1 and 2 are the priority. If anything takes longer than expected, it comes from Milestone 3 and the stretch goals. I will not, In Sha Allah, start adding new types until the stability work is done, because debugging type bugs on top of GC bugs is very difficult.

**Note:** All new tests will, In Sha Allah, be run on PostgreSQL 14 through 17 to make sure the extension works on all supported versions.

---

## Contributions

- [PR #13](https://github.com/pljulia/pljulia/pull/13) Fix build compatibility with Julia 1.9+ and PostgreSQL 14/15: Merged
- [PR #17](https://github.com/pljulia/pljulia/pull/17) Fix buffer overflow when loading Julia packages with long names: Open
- [PR #21](https://github.com/pljulia/pljulia/pull/21) Add return_bool to regression test suite and remove duplicate test file: Open
- [Issue #12](https://github.com/pljulia/pljulia/issues/12) Build fails on Julia 1.12.4 with PostgreSQL 16 on Linux (Ubuntu): Reported and investigated
- [Issue #22](https://github.com/pljulia/pljulia/issues/22) Deprecated Pkg.installed() should be replaced by Pkg.dependencies(): Reported
- [Issue #33](https://github.com/pljulia/pljulia/issues/33) 5 of 24 regression tests fail on PostgreSQL 16 due to hidden symbol visibility: Reported and diagnosed

**Note:** My focus so far has been on reading the codebase, testing across versions, and identifying issues rather than submitting many PRs. At this stage of PL/Julia, finding and documenting the problems is more valuable than rushing fixes — it builds the roadmap that the GSoC work will follow.

---

## About Me

I'm Abdelsalam Mostafa, a Computer Science graduate and Software Engineer from Egypt. I got into PostgreSQL internals after taking [CMU Intro to Databases](https://15445.courses.cs.cmu.edu/) and [Amr El Helw's Database Internals course](https://github.com/aelhelw/techvault/blob/main/Relational_Databases_Viewing_Order.md) (ex SE manager at Meta, MongoDB, Google). These courses made me want to contribute to PostgreSQL, which led me to PL/Julia.

- **C/C++ and Problem Solving:** I'm a competitive programmer. I solved a lot of problems using C++ and qualified to ACPC (Africa and Arab Collegiate Programming Contest), a regional contest of [ICPC](https://icpc.global/). This gave me strong experience with C, memory management, and debugging, which are all needed for working on a PostgreSQL C extension.
- **Databases:** I studied databases from multiple sources: my university courses, CMU 15-445, and Amr El Helw's course. I understand how query processing, storage engines, ..
- **CS Fundamentals:** I graduated with very good grades. I studied Operating Systems, Computer Architecture, Data Structures, Algorithms, and more. These helped me understand how PL/Julia interacts with PostgreSQL at a low level.
- **Julia:** I'm not an expert yet, but I have been learning it since I started contributing to PL/Julia. I read the [Julia embedding docs](https://docs.julialang.org/en/v1/manual/embedding/) and I understand how the C API works from working on my PRs.
- **Open Source:** My first contribution was to [Manim](https://github.com/ManimCommunity/manim), an animation library I was using. I found an issue, fixed it, and it got merged. After that I started looking for bigger projects, which is how I found PL/Julia.

---

## Commitment

I will work full-time on the project during the coding period, In Sha Allah.
I will dedicate 20-35 hours weekly, In Sha Allah.

---

## References

- PL/Julia project: https://gitlab.com/pljulia/pljulia
- Julia embedding documentation: https://docs.julialang.org/en/v1/manual/embedding/
- PostgreSQL SPI documentation: https://www.postgresql.org/docs/current/spi.html
- PostgreSQL PL Matrix: https://wiki.postgresql.org/wiki/PL_Matrix
- Julia C API stability discussion: https://github.com/JuliaLang/julia/issues/56805
- JuliaC.jl: https://github.com/JuliaLang/JuliaC.jl
