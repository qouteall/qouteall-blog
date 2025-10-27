---
date: 2025-08-03
tags:
  - Programming
---
# Traps to Developers

A summarization of some traps to developers. There traps are unintuitive things that are easily misunderstood and cause bugs.

<!-- truncate -->

This article spans a wide range of knowledge. If you find a mistake or have a suggestion, please leave a comment in [GitHub discussion](https://github.com/qouteall/qouteall-blog/discussions).

## Summarization of traps

### HTML and CSS

- `min-width` is `auto` by default. Inside flexbox or grid, `min-width: auto` often makes min width determined by content. It has higher priority than many other CSS attributes including `flex-shrink`, `width: 0` and `max-width: 100%`. It's recommended to set `min-width: 0`. [See also](https://developer.mozilla.org/en-US/docs/Web/CSS/min-width)
- Horizontal and vertical are different in CSS:
  - Normally `width: auto` tries fill available space in parent. But `height: auto` normally tries to just expand to fit content.
  - For inline elements, inline-block elements and float elements, `width: auto` does not try to expand.
  - `margin: 0 auto` centers horizontally. But `margin: auto 0` normally become `margin: 0 0` which does not center vertically. In a flexbox with `flex-direction: column`, `margin: auto 0` can center vertically.
  - Margin collapse happens vertically but not horizontally.
  - The above flips when layout direction flips (e.g. `writing-mode: vertical-rl`)
- Block formatting context (BFC):
  - `display: flow-root` creates a BFC.
    (There are other ways to create BFC, like `overflow: hidden`, `overflow: auto`, `overflow: scroll`, `display:table`, but with side effects)
  - Margin collapse. Two vertically touching siblings can overlap margin. Child margin can "leak" outside of parent. Margin collapse can be avoided by BFC. Margin collapse also doesn't happen when `border` or `padding` spcified
  - If a parent only contains floating children, the parent's height will collapse to 0. Can be fixed by BFC.
- [Stacking context](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_positioned_layout/Stacking_context).
  
  In these cases, it will start a new stacking context:
  
  - The attributes that give special rendering effects (`transform`, `filter`, `perspective`, `mask`, `opacity` etc.) will create a new stacking context
  - `position: fixed` or `position: sticky` creates a stacking context
  - Specifies `z-index` and `position` is `absolute` or `relative`
  - Specifies `z-index` and the element is inside flexbox or grid
  - `isolation: isolate`
  - ...
  
   Stacking context can cause these behaviors:
  
  - `z-index` doesn't work across stacking contexts. It only works within a stacking context.
  - Stacking context can affect the coordinate of `position: absolute` or `fixed`. (The underlying logic is complex, [see also](https://developer.mozilla.org/en-US/docs/Web/CSS/position) )
  - `position: sticky` doesn't work across stacking context.
  - `overflow: visible` will still be clipped by stacking context
  - `background-attachment: fixed` will position based on stacking context

- On mobile browsers, the top address bar and bottom navigation bar can go out of screen when you scroll down. `100vh` correspond to the height when top bar and bottom bar gets out of screen, which is larger than the height when the two bars are on screen. The modern solution is `100dvh`.
- `width: 100vw` makes the width-that-excludes-scrollbar to be `100vw`, which can make the total width (including scrollbar) to horizontally overflow. `width: 100%` can avoid that issue.
- `position: absolute` is not based on its parent. It's based on its nearest positioned ancestor (the nearest ancestor that has `position` be `relative`, `absolute` or creates stacking context).
- [`backdrop-filter: blur` does not consider ambient things](https://www.joshwcomeau.com/css/backdrop-filter/#the-issue).
- If the parent's `display` is `flex` or `grid`, then the child's `float` has no effect
- If the parent's width/height is not pre-determined, then percent width/height (e.g. `width: 50%`, `height: 100%`) doesn't work. (It avoids circular dependency where parent height is determined by content height, but content height is determined by parent height.)
- `display: inline` ignores `width` `height` and `margin-top` `margin-bottom`
- Whitespace collapse. [See also](https://blog.dwac.dev/posts/html-whitespace/)
  - By default, newlines in html are treated as spaces. Multiple spaces together collapse into one. 
  - `<pre>` can avoid collapsing whitespace but has weird behavior in the beginning and end of content.
  - Often the spaces in the beginning and end of content are ignored, but this doesn't happen in `<a>`.
  - Any space or line break between two `display: inline-block` elements will be rendered as spacing. This doesn't happen in flexbox or grid.
- `text-align` aligns text and inline things, but doesn't align block elements (e.g. normal divs).
- By default `width` and `height` doesn't include padding and border. `width: 100%` with `padding: 10px` can still overflow the parent. `box-sizing: border-box` make the width/height include border and padding.
- [Cumulative Layout Shift](https://web.dev/articles/cls). It's recommended to specify `width` and `height` attribute in `<img>` to avoid layout shift due to image loading delay.
- File download request is not shown in Chrome dev tool, because it only shows networking in current tab, but file download is treated as in another tab. To inspect file download request, use `chrome://net-export/`.
- JS-in-HTML may interfere with HTML parsing. For example `<script>console.log('</script>')</script>` makes browser treat the first `</script>` as ending tag. [See also](https://sirre.al/2025/08/06/safe-json-in-script-tags-how-not-to-break-a-site/)

### Unicode and text

- Two concepts: code point, grapheme cluster:
  - Grapheme cluster is the "unit of character" in GUI.
  - For visible ascii characters, a character is a code point, a character is a grapheme cluster.
  - An emoji is a grapheme cluster, but it may consist of many code points.
  - In UTF-8, a code point can be 1, 2, 3 or 4 bytes. The byte number does not necessarily represent code point number.
  - In UTF-16, each UTF-16 code unit is 2 bytes. A code point can be 1 code unit (2 bytes) or 2 code units (4 bytes, surrogate pair).
  - JSON string `\u` escape uses surrogate pair. `"\uD83D\uDE00"` in JSON has only one code point.
- Different in-memory string behaviors in different languages:
  - Rust use UTF-8 for in-memory string. `s.len()` gives byte count. Rust does not allow directly indexing on a `str` (but allows subslicing). `s.chars().count()` gives code point count. Rust is strict in UTF-8 code point validity (for example Rust doesn't allow subslice to cut on invalid code point boundary).
  - Java, C# and JS's string encoding is similar to UTF-16 [^string_encoding]. String length is code unit count. Indexing works on code units. Each code unit is 2 bytes. One code point can be 1 code unit or 2 code units.
  - In Python, `len(s)` gives code point count. Indexing gives a string that contains one code point.
  - Golang string has no constraint of encoding and is similar to byte array. String length and indexing works same as byte array. But the most commonly used encoding is UTF-8. [See also](https://go.dev/blog/strings)
  - In C++, `std::string` has no constraint of encoding and is similar to byte array. String length and indexing is based on bytes.
  - No language mentioned above do string length and indexing based on grapheme cluster.
  - In SQL, `varchar(100)` limits 100 code points (not byte).
- Some text files have byte order mark (BOM) at the beginning. For example, FE FF means file is in big-endian UTF-16. EF BB BF means UTF-8. It's mainly used in Windows. Some non-Windows software does not handle BOM.
- When converting binary data to string, often the invalid places are replaced by � (U+FFFD)
- [Confusable characters](https://github.com/unicode-org/icu/blob/main/icu4c/source/data/unidata/confusables.txt).
- Normalization. For example é can be U+00E9 (one code point) or U+0065 U+0301 (two code points). String comparision works on binary data and don't consider normalization.
- [Zero-width characters](https://ptiglobal.com/the-beauty-of-unicode-zero-width-characters/), [Invisible characters](https://invisible-characters.com/)
- Line break. Windows often use CRLF `\r\n` for line break. Linux and MacOS often use LF `\n` for line break.
- Locale ([elaborated below](#locale)).


[^string_encoding]: Strictly speaking, they use [WTF-16](https://simonsapin.github.io/wtf-8/#ill-formed-utf-16) encoding, which is similar to UTF-16 but allows invalid surrogate pairs. Also, Java has an optimization that use Latin-1 encoding (1 byte per code point) for in-memory string if possible. But the API of `String` still works on WTF-16 code units. Similar things may happen in C# and JS. 

### Floating point

- NaN. Floating point NaN is not equal to any number including itself. NaN == NaN is always false (even if the bits are same). NaN != NaN is always true. Computing on NaN usually gives NaN (it can "contaminate" computation).
- There are +Inf and -Inf. They are not NaN.
- There is a negative zero -0.0 which is different to normal zero. The negative zero equals zero when using floating point comparision. Normal zero is treated as "positive zero". The two zeros behave differently in some computations (e.g. `1.0 / 0.0 == Inf`, `1.0 / -0.0 == -Inf`, `log(0.0) == -Inf`, `log(-0.0)` is NaN)
- JSON standard doesn't allow NaN or Inf:
  - JS `JSON.stringify` turns NaN and Inf to null.
  - Python `json.dumps(...)` will directly write `NaN`, `Infinity` into result, which is not compliant to JSON standard. `json.dumps(..., allow_nan=False)` will raise `ValueError` if has NaN or Inf.
  - Golang `json.Marshal` will give error if has NaN or Inf.
- Directly compare equality for floating point may fail due to precision loss. Compare equality by things like `abs(a - b) < 0.00001`
- JS use floating point for all numbers. The max "safe" integer is $2^{53}-1$. The "safe" here means every integer in range can be accurately represented. Outside of the safe range, most integers will be inaccurate. For large integer it's recommended to use `BigInt`.
  
  If a JSON contains an integer larger than that, and JS deserializes it using `JSON.parse`, the number in result will be likely inaccurate. The workaround is to use other ways of deserializing JSON or use string for large integer. 
  
  (Putting millisecond timestamp integer in JSON fine, as millisecond timestamp exceeds limit in year 287396. But nanosecond timestamp suffers from that issue.)
- Associativity law and distribution law doesn't strictly hold because of precision loss. Parallelizing matrix multiplication and sum dynamically using these laws can be non-deterministic. See also: [Defeating Nondeterminism in LLM Inference](https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/)
- Division is much slower than multiplication (unless using approximation). Dividing many numbers with one number can be optimized by firstly computing reciprocal then multiply by reciprocal.
- These things can make different hardware have different floating point computation results:
  - Hardware FMA (fused multiply-add) support. `fma(a, b, c) = a * b + c` (in some places `a + b * c`). Most modern hardware make intermediary result in FMA have higher precision. Some old hardware or embedded processors don't do that and treat it as normal multiply and add.
  - Floating point has a [Subnormal range](https://en.wikipedia.org/wiki/Subnormal_number) to make very-close-to-zero numbers more accurate. Most mondern hardware can handle them, but some old hardware and embedded processors treat subnormals as zero.
  - Rounding mode. The standard allows different rounding modes like round-to-nearest-ties-to-even (RNTE) or round-toward-zero (RTZ). 
    - In X86 and ARM, rounding mode is thread-local mutable state can be set by special instructions. It's not recommended to touch the rounding mode as it can affect other code.
    - In GPU, there is no mutable state for rounding mode. Rasterization often use RNTE rounding mode. In CUDA different rounding modes are associated by different instructions.
  - Math functions (e.g. sin, log) may be less accurate in some embedded hardware or old hardware.
  - X86 has legacy FPU which has 80-bit floating point registers and per-core rounding mode state. It's recommended to not use them.
  - ......
- Floating point accuracy is low for values with very large absolute value or values very close to zero. It's recommended to avoid temporary result to have very large absolute value or be very close-to-zero.
- Iteration can cause error accumulation. For example, if something need to rotate 1 degree every frame, don't cache the matrix and multiply 1-degree rotation matrix every frame. Compute angle based on time then re-calculate rotation matrix from angle.


### Time

- [Leap second](https://en.wikipedia.org/wiki/Leap_second). Unix timestamp is "transparent" to leap second, which means converting between Unix timestamp and UTC time ignores leap second. A common solution is leap smear: make the time measured in Unix timestamp stretch or squeeze near a leap second.
- Time zone. UTC and Unix timestamp is globally uniform. But human-readable time is time-zone-dependent. It's recommended to store timestamp in database and convert to human-readable time in UI, instead of storing human-readable time in database.
- Daylight Saving Time (DST): In some region people adjust clock forward by one hour in warm seasons.
- Time may "go backward" due to NTP sync.
- It's recommended to configure the server's time zone as UTC. Different nodes having different time zones will cause trouble in distributed system. After changing system time zone, the database may need to be reconfigured or restarted.
- There are two clocks: hardware clock and system clock. The hardware clock itself doesn't care about time zone. Linux treats it as UTC by default. Windows treats it as local time by default.


### Java

- `==` compares object reference. Should use `.equals` to compare object content. 
- Forget to override `equals` and `hashcode`. It will use object identity equality by default in map key and set.
- Mutate the content of map key object (or set element object) makes the container malfunciton.
- A method that returns `List<T>` may sometimes return mutable `ArrayList`, but sometimes return `Collections.emptyList()` which is immutable. Trying to modify on `Collections.emptyList()` throws `UnsupportedOperationException`.
- A method that returns `Optional<T>` may return `null` (this is not recommended, but this do exist in real codebases).
- Null is ambiguous. If `get()` on a map returns null, it may be either value is missing or value exists but it's null (can distinguish by `containsKey`). Null field and missing field in JSON are all mapped to null in Java object. [See also](https://committing-crimes.com/articles/2025-09-16-null-and-absence/)
- Implicitly converting `Integer` `Long` `Double` etc. to `int` `long` `double` etc. can cause `NullPointerException`.
- Return in `finally` block swallows any exception thrown in the `try` or `catch` block. The method will return the value from `finally`.
- Interrupt. Some libraries ignore interrupt. If a thread is interrupted and then load a class, and class initialization has IO, then class may fail to load.
- Thread pool does not log exception of tasks sent by `.submit()` by default. You can only get exception from the future returned by `.submit()`. Don't discard the future. And `scheduleAtFixedRate` task silently stop if exception is thrown.
- Literal number starting with 0 will be treated as octal number. (`0123` is 83)
- When debugging, debugger will call `.toString()` to local variables. Some class' `.toString()` has side effect, which cause the code to run differently under debugger. This can be disabled in IDE.
- Before [Java24](https://openjdk.org/jeps/491) virtual thread can be "pinned" when blocking on `synchronized` lock, which may cause deadlock. It's recommended to upgrade to Java 24 if you use virtual thread.
- It's not recommended to override `finalize()`. If `finalize()` runs too slow, it can block GC. Exceptions out of `finalize()` are not logged. A dead object can resurrect itself in `finalize()`, and if a resurrected object become dead again, `finalize()` won't be called again. Use [`Cleaner`](https://docs.oracle.com/javase/9/docs/api/java/lang/ref/Cleaner.html) for GC-directed disposal.

### Golang

- `append()` reuses memory region if capacity allows. Appending to a subslice can overwrite parent if they share memory region.
- `defer` executes when the function returns, not when the lexical scope exits.
- `defer` capture mutable variable.
- About `nil`:
  - There are nil slice and empty slice (the two are different). But there is no nil string, only empty string. The nil map can be read like an empty map, but nil map cannot be written.
  - Interface `nil` weird behavior. Interface pointer is a fat pointer containing type info and data pointer. If the data pointer is null but type info is not null, then it will not equal `nil`.
- Before Go 1.22, [loop variable capture issue](https://go.dev/blog/loopvar-preview).
- Dead wait. [Understanding Real-World Concurrency Bugs in Go](https://songlh.github.io/paper/go-study.pdf)
- Different kinds of timeout. [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)
- Having interior pointer to an object keeps the whole object alive. This may cause memory leak.


### C/C++

- Storing a pointer of an element inside `std::vector` and then grow the vector, `vector` may re-allocate content, making element pointer no longer valid.
- `std::string` created from literal string may be temporary. Taking `c_str()` from a temporary string is wrong.
- [Iterator invalidation](https://learnmoderncpp.com/2024/09/04/understanding-iterator-invalidation/). Modifying a container when looping on it.
- `std::views::filter` malfunctions when element is mutated that predicate result changes in multi-pass iteration. [See also](https://github.com/CppCon/CppCon2024/blob/main/Presentations/Taming_the_Cpp_Filter_View.pdf). `std::views::as_rvalue` with `std::ranges::to` mutates the element which can trigger that issue. [See also](https://github.com/philsquared/cpponsea2025-slides/blob/main/Presentations/Faster_Safer_Better_Ranges.pdf)
- `std::remove` doesn't remove but just rearrange elements. `erase` actually removes.
- Literal number starting with 0 will be treated as octal number. (`0123` is 83)
- Destructing a deep tree structure can stack overflow. Solution is to replace recursion with loop in destructor.
- Undefined behaviors. The compiler optimization aim to keep defined behavior the same, but can freely change undefined behavior. Relying on undefined behavior can make program break under optimization. [See also](https://russellw.github.io/undefined-behavior)
  - Accessing uninitialized memory is undefined behavior. Converting a `char*` to struct pointer can be seen as accessing uninitialized memory, because the object lifetime hasn't started. It's recommended to put the struct elsewhere and use `memcpy` to initialize it.
  - Accessing invalid memory (e.g. null pointer) is undefined behavior.
  - Integer overflow/underflow is undefined behavior. Note that unsigned integer can underflow below 0.
  - Aliasing.
    - Aliasing means multiple pointers point to the same place in memory.
    - Strict aliasing rule: If there are two pointers with type `A*` and `B*`, then compiler assumes two pointer can never equal. If they equal, it's undefined behavior. Except in two cases: 1. `A` and `B` has subtyping relation 2. converting pointer to byte pointer (`char*`, `unsigned char*` or `std::byte*`) (the reverse does not apply).
    - Pointer provenance. Two pointers from two different provenances are treated as never alias. If their address equals, it's undefined behavior. [See also](https://www.ralfj.de/blog/2020/12/14/provenance.html)
- Alignment.
  - For example, 64-bit integer's address need to be disivible by 8. In ARM, accessing memory in unaligned way can cause crash.
  - Unaligned memory access is undefined behavior.
  - Directly treating a part of byte buffer as a struct is undefined behavior. Not only due to alignment, but also due to object lifetime not yet started [^start_object_lifetime].
  - Alignment can cause padding in struct that waste space.
  - Some SIMD instructions only work with aligned data. For example, AVX instructions usually require 32-byte alignment.
- Global variable initialization runs before `main`. [Static Initialization Order Fiasco](https://en.cppreference.com/w/cpp/language/siof.html).

[^start_object_lifetime]: Directly treating existing binary data as struct is undefined behavior because the object lifetime hasn't started. But using `memcpy` to initialize a struct is fine.

### Python

- Default argument is a stored value that will not be re-created on every call.
- Be careful about indentation when copying and pasting Python code.

### SQL Databases

- Null is special. 
  - `x = null` doesn't work. `x is null` works. Null does not equal itself, similar to NaN.
  - Unique index allows duplicating null (except in Microsoft SQL server). 
  - `select distinct` may treat nulls as the same (this is database-specific). 
  - `count(x)` and `count(distinct x)` ignore rows where `x` is null.
- Date implicit conversion can be timezone-dependent.
- Complex join with disctinct may be slower than nested query. [See also](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/dont-use-distinct-as-a-join-fixer/)
- In MySQL (InnoDB), if string field doesn't have `character set utf8mb4` then it will error if you try to insert a text containing 4-byte UTF-8 code point.
- MySQL (InnoDB) default to case-insensitive.
- MySQL (InnoDB) can do implicit conversion by default. `select '123abc' + 1;` gives 124.
- MySQL (InnoDB) gap lock may cause deadlock.
- In MySQL (InnoDB) you can select a field and group by another field. It gives nondeterministic result.
- In SQLite the field type doesn't matter unless the table is `strict`.
- SQLite by default does not do vacuum. The file size only increases and won't shrink. To make it shrink you need to either manually `vacuum;` or enable `auto_vacuum`.
- Foreign key may cause implicit locking, which may cause deadlock.
- Locking may break repeatable read isolation (it's database-specific).
- Distributed SQL database may doesn't support locking or have weird locking behaviors. It's database-specific.
- If the backend has N+1 query issue, the slowness may won't be shown in slow query log, because the backend does many small queries serially and each individual query is fast.
- Long-running transaction can cause problems (e.g. locking). It's recommended to make all transactions finish quickly.
- If a string column is used in index or primary key, it will have length limit. MySQL applies the limitation when changing table schema. PostgreSQL applies the limitation by erroring when inserting or updating data.
- Whole-table locks that can make the service temporarily unusable:
  - In MySQL (InnoDB) 8.0+, adding unique index or foreign key is mostly concurrent (only briefly lock) and won't block operations. But in older versions it may do whole-table lock.
  - `mysqldump` used without `--single-transaction` cause whole-table read lock.
  - In PostgreSQL, `create unique index` or `alter table ... add foreign key` cause whole-table read-lock. To avoid that, use `create unique index concurrently` to add unique index. For foreign key, use `alter table ... add foreign key ... not valid;` then `alter table ... validate constraint ...`.
- About ranges:
  - If you store non-overlapping ranges, querying the range containing a point by `select ... from ranges where p >= start and p <= end` is inefficient (even when having composite index of `(start, end)`). An efficient way: `select * from (select ... from ranges where start <= p order by start desc limit 1) where end >= p` (only require index of `start` column).
  - For overlappable ranges, normal B-tree index is not sufficient for efficient querying. It's recommended to use spatial index in MySQL and GiST in PostgreSQL.




### Concurrency and Parallelism

- `volatile`:
  - `volatile` itself cannot replace locks. `volatile` itself doesn't provide atomicity.
  - You don't need `volatile` for data protected by lock. Locking can already establish memory order and prevent some wrong optimizations.
  - In C/C++, `volatile` only avoids some wrong optimizations, and won't automatically add memory barrier instruction for `volatile` access.
  - In Java, `volatile` accesses have sequentially-consistent ordering (JVM will use memory barrier instruction if needed)
  - In C#, accesses to the same `volatile` value have release-aquire ordering (CLR will use memory barrier instruction if needed)
  - `volatile` can avoid wrong optimization related to reordering and merging memory reads/writes. (Compiler can merge reads by caching a value in register. Compiler can merge writes by only writing to register and delaying writing to memory. A read after a write can be optimized out.).
- Time-of-check to time-of-use ([TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use)).
- In SQL database, for special uniqueness constraints that doesn't fit simple unique index (e.g. unique across two tables, conditional unique, unique within time range), if the constraint is enforced by application, then:
  - In MySQL (InnoDB), if in repeatable read level, application checks using `select ... for update` then insert, and the unique-checked column has index, then it works due to gap lock. (Note that gap lock may cause deadlock under high concurrency, ensure deadlock detection is on and use retrying).
  - In PostgreSQL, if in repeatable read level, application checks using `select ... for update` then insert, it's not sufficient to enforce constraint under concurrency (due to write skew). Some solutions:
    - Use serializable level
    - Don't rely on application to enforce constraint:
      - For conditional unique, use partial unique index. 
      - For uniqueness across two tables case, insert redundant data into one extra table with unique index.
      - For time range exclusiveness case, use range type and exclude constraint.
- Atomic reference counting (`Arc`, `shared_ptr`) can be slow when many threads frequently change the same counter. [See also](https://pkolaczk.github.io/server-slower-than-a-laptop/)
- About read-write lock: trying to write lock when holding read lock can deadlock. The correct way is to firstly release the read lock, then acquire write lock, and the conditions that were checked in read lock need to be re-checked.
- Reentrant lock:
  - Reentrant means one thread can lock twice (and unlock twice) without deadlocking. Java's `synchronized` and `ReentrantLock` are reentrant.
  - Non-reentrant means if one thread lock twice, it will deadlock. Rust `Mutex` and Golang `sync.Mutex` are not reentrant.
- [False sharing](https://en.wikipedia.org/wiki/False_sharing) of the same cache line costs performance.

### Common in many languages

- Forget to check for null/None/nil.
- Modifying a container when for looping on it. Single-thread "data race".
- Unintended sharing of mutable data. For example in Python `[[0] * 10] * 10` does not create a proper 2D array.
- For non-negative integer `(low + high) / 2` may overflow. A safer way is `low + (high - low) / 2`.
- Short circuit. `a() || b()` will not run `b()` if `a()` returns true. `a() && b()` will not run `b()` when `a()` returns false.
- When using profiler: the profiler may by default only include CPU time which excludes waiting time. If your app spends 90% time waiting on database, the flamegraph may not include that 90% which is misleading.
- Try to cancel some async operation, but the callback still runs.
- Assertion should not be used for validating data. Should do proper error handling to invalid external data. Assertion should check internal invariants.

### Linux and bash

- If the current directory is moved, `pwd` still shows the original path. `pwd -P` shows the real path.
- `cmd > file 2>&1` make both stdout and stderr go to file. But `cmd 2>&1 > file` only make stdout go to file but don't redirect stderr.
- File name is case sensitive (unlike Windows).
- There is a capability system for executables, apart from file permission sytem. Use `getcap` to see capability.
- Unset variables. If `DIR` is unset, `rm -rf $DIR/` becomes `rm -rf /`. Using `set -u` can make bash error when encountering unset variable.
- If you want a script to add variables and aliases to current shell, it should be executed by using `source script.sh`, instead of directly executing. But the effect of `source` is not permanent and doesn't apply after re-login. It can be made permanent by putting into `~/.bashrc`.
- Bash has caching between command name and file path of command. If you move one file in `$PATH` then using that command gives ENOENT. Refresh cache using `hash -r`
- Using a variable unquoted will make its line breaks treated as space.
- `set -e` can make the script exit immediately when a sub-command fails, but it doesn't work inside function whose result is condition-checked (e.g. the left side of `||`, `&&`, condition of `if`). [See also](https://stratus3d.com/blog/2019/11/29/bash-errexit-inconsistency/)

### Backend-related

- K8s `livenessProbe` used with debugger. Breakpoint debugger usually block the whole application, making it unable to respond health check request, so it can be killed by K8s `livenessProbe`.
- Don't use `:latest` image. They can change at any time.
- In Redis, getting keys by a prefix `KEYS prefix-*` is a slow operation that will traverse all keys. Use Redis hash map for that use case.

### React

- Modify state in rendering code.
- React compares equality using reference equality, not content equality.
  - The objects and arrays that are newly created in rendering are treated as always-new. Use `useMemo` to fix.
  - The closure functions that are created in rendering are also always-new. Use `useCallback` to fix.
  - If an always-new thing is put into `useEffect` dependency array, the effect will run on every render. See also [Cloudflare indicent 2025 Sept-12](https://blog.cloudflare.com/deep-dive-into-cloudflares-sept-12-dashboard-and-api-outage/). 
  - Don't forget to include dependencies in the dependency array. And the dependencies also need to be memoed.
- When using effect to manage `setInterval` `removeInterval`, if the effect has dependency value, it will remove timer and re-add timer when dependency changes, which can mess up the timing.
- State values themselves should be immutable. Don't directly set fields of state objects. Always recreate whole object.
- Forget to include value in `useEffect` dependency array.
- Forget clean up in `useEffect`.
- Closure trap. Closure can capture a state. If the state changes, the closure still captures the old state. 
  - One solution is to make closure not capture state and access state within `useReducer`. 
  - Another solution is to put mutable thing in `useRef` (note that changing value inside ref don't trigger re-rendering, you need to change state or prop to trigger re-rendering)
- `useEffect` firstly runs in next iteration of event loop, after browser renders [^render_ambiguity] the web page. Doing initialization in `useEffect` is not early enough and may cause visual flicker. Use `useLayoutEffect` for early initialization (it will run in the same iteration of event loop). When using ref to get DOM object, it won't be accessible during first rendering (component function call). It can be accessed in `useLayoutEffect`.

[^render_ambiguity]: In web dev the word "render" is ambiguous. When browser "renders" it draws content to present in screen. When the React component "renders" the component function is called, not drawing things to screen.

### Git

- Rebase can rewrite history. After rebasing local branch, normal push will give weird result (because history is rewritten). Rebase should be used with force push. If remote branch's history is rewritten, pulling should use `--rebase`.
  - Force pushing with `--force-with-lease` can sometimes avoid overwriting other developers' commits. But if you fetch then don't pull, `--force-with-lease` cannot protect.
- Reverting a merge doesn't fully cancel the side effect of the merge. If you merge B to A and then revert, merging B to A again has no effect. One solution is to revert the revert of merge. (A cleaner way to cancel a merge, instead of reverting merge, is to backup the branch, then hard reset to commit before merge, then cherry pick commits after merge, then force push.)
- In GitHub, if you accidentally commited secret (e.g. API key) and pushed to public, even if you override it using force push, GitHub will still record that secret. [See also](https://trufflesecurity.com/blog/guest-post-how-i-scanned-all-of-github-s-oops-commits-for-leaked-secrets) [Example activity tab](https://github.com/SharonBrizinov/test-oops-commit/compare/e6533c7bd729957b2eb31e88065c5158d1317c5e...9eedfa00983b7269a75d76ec5e008565c2eff2ef)
- In GitHub, if there is a private repo A and you forked it as B (also private), then when A become public, the private repo B's content is also publicly accessible, even after deleting B. [See also](https://trufflesecurity.com/blog/anyone-can-access-deleted-and-private-repo-data-github).
- GitHub by default allows deleting a release tag, and adding a new tag with same name, pointing to another commit. It's not recommended to do that. Many build systems cache based on release tag, which breaks under that. It can be disabled in rulesets configuration.
- `git stash pop` does not drop the stash if there is a conflict.
- In Windows, Git often auto-convert cloned text files to be CRLF line ending. But in WSL many software (e.g. bash) doesn't work with files with CRLF. Using `git clone --config core.autocrlf=false -c core.eol=lf ...` can make git clone as LF.
- MacOS auto adds `.DS_Store` files into every folder. It's recommended to add `**/.DS_Store` into `.gitignore`.

### Networking

- Some routers and firewall silently kill idle TCP connections without telling application. Some code (like HTTP client libraries, database clients) keep a pool of TCP connections for reuse, which can be silently invalidated (using these TCP connection will get RST). To solve it, configure system TCP keepalive. [See also](https://tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/)
  - Note that [HTTP/1.0 Keep-Alive](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Keep-Alive) is different to TCP keepalive.
- The result of `traceroute` is not reliable. [See also](https://gekk.info/articles/traceroute.htm). Sometimes [tcptraceroute](https://linux.die.net/man/1/tcptraceroute) is useful.
- TCP slow start can increase latency. Can be fixed by disabling `tcp_slow_start_after_idle`. [See also](https://ably.com/blog/optimizing-global-message-transit-latency-a-journey-through-tcp-configuration)
- TCP sticky packet. Nagle's algorithm delays packet sending. It will increase latency. Can be fixed by enabling `TCP_NODELAY`. [See also](https://brooker.co.za/blog/2024/05/09/nagle.html) 
- If you put your backend behind Nginx, you need to configure connection reuse, otherwise under high concurrency, connection between nginx and backend may fail, due to not having enough internal ports.
- Nginx `proxy_buffering` delays SSE.
- The HTTP protocol does not explicitly forbit GET and DELETE requests to have body. Some places do use body in GET and DELETE requests. But many libraries and HTTP servers does not support them.
- One IP can host multiple websites, distinguished by domain name. The HTTP header `Host` and SNI in TLS handshake carries domain name, which are important. Some websites cannot be accessed via IP address.
- CORS (cross-origin resource sharing). For requests to another website (origin), the browser will prevent JS from getting response, unless the server's response contains header `Access-Control-Allow-Origin` and it matches client website. This requires configuring the backend. If you want to pass cookie to another website it involves more configuration.
  
  Generally, if your frontend and backend are in the same website (same domain name and port) then there is no CORS issue.
- [Reverse path filtering](https://en.wikipedia.org/wiki/Reverse-path_forwarding). When routing is asymmetric, packet from A to B use different interface than packets from B to A, then reverse path filtering rejects valid packets.
- In old versions of Linux, if `tcp_tw_recycle` is enabled, it aggressively recycles connection based on TCP timestamp. NAT and load balancer can make TCP timestamp not monotonic, so that feature can drop normal connections.

### Locale

- The upper case and lower case can be different in other natural languages. In Turkish (tr-TR) lowercase of `I` is `ı` and upper case of `i` is `İ`. The `\w` (word char) in regular expression can be locale-dependent.
- Letter ordering can be different in other natural languages. Regular expression `[a-z]` may malfunction in other locale.
- Text notation of floating-point number is locale-dependent. `1,234.56` in US correspond to `1.234,56` in Germany.
- CSV use normally use `,` as spearator, but use `;` as separator in German locale.
- [Han unification](https://en.wikipedia.org/wiki/Han_unification). Some characters in different language with slightly different appearance use the same code point. Usually a font will contain variants for different languages that render these characters differently. [HTML code](https://github.com/qouteall/qouteall-blog/blob/main/blog/2025/unicode-unification-example.html) ![](unicode_unification_example.png)

### Regular expression

- Regular expression cannot parse the syntax that allows infinite nesting (because regular expression engine use finite state machine. Infinite nesting require infinite states to parse). HTML allows infinite nesting. But it's ok to use regex to parse HTML of a specific website.
- Regular expression behavior can be locale-dependent (depending on which regular expression engine).
- There are many different "dialects" of regular expression. Don't assume a regular expression that works in JS can work in Java.
- A separate regular expression validation can be out-of-sync with actual data format. [Crowdstrike incident](https://www.crowdstrike.com/wp-content/uploads/2024/08/Channel-File-291-Incident-Root-Cause-Analysis-08.06.2024.pdf) was caused by a wrong separate regular expression validation. It's recommended to **avoid separate regular expression validation. Reuse parsing code for validation**. See also: [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- Backtracking performance issue. See also: [Cloudflare indicent 2019 July-2](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/), [Stack Exchange incident 2016 July-20](https://stackstatus.tumblr.com/post/147710624694/outage-postmortem-july-20-2016)

### Other

- YAML:
  - YAML is space-sensitive, unlike JSON. `key:value` is wrong. `key: value` is correct.
  - YAML doesn't allow using tab for indentation.
  - [Norway country code `NO` become false if unquoted](https://www.bram.us/2022/01/11/yaml-the-norway-problem/).
  - [Git commit hash may become number if unquoted](https://tmendez.dev/posts/rng-git-hash-bug/).
  - [The yaml document from hell](https://ruudvanasseldonk.com/2023/01/11/the-yaml-document-from-hell)
- When using Microsoft Excel to open a CSV file, Excel will do a lot of conversions, such as date conversion (e.g. turn `1/2` and `1-2` into `2-Jan`) and Excel won't show you the original string. [The gene SEPT1 was renamed due to this Excel issue](https://en.wikipedia.org/wiki/SEPTIN1). Excel will also make large numbers inaccurate (e.g. turn `12345678901234567890` into `12345678901234500000`) and won't show you the original accurate number, because Excel internally use floating point for number.
- It's recommended to configure billing limit when using cloud services, especially serverless. See also: [ServerlessHorrors](https://serverlesshorrors.com/)
- Big endian and little endian in binary file and net packet.


