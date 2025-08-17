---
date: 2025-08-03
tags:
  - Programming
---
# Traps to Developers

A summarization of some traps to developers. There traps are unintuitive things that are easily misunderstood and cause bugs.

<!-- truncate -->

## Summarization of traps

### HTML and CSS

- Inside flexbox or grid, `min-width` is `auto` by default. `min-width: auto` makes min width determined by content. It has higher priority than many other CSS attributes including `flex-shrink`, `overflow: hidden`, `width: 0` and `max-width: 100%`. It's recommended to set `min-width: 0`.
- Horizontal and vertical are different in CSS:
  - Normally `width: auto` tries fill available space in parent. But `height: auto` normally tries to just expand to fit content.
  - For inline elements, inline-block elements and float elements, `width: atuo` does not try to expand.
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
  - `position: absolute` or `fixed` use coordinate based on nearest positioned ancestor. Stacking context is positioned, which can affect that.
  - `position: sticky` doesn't work across stacking context.
  - `overflow: visible` will still be clipped by stacking context
  - `background-attachment: fixed` will position based on stacking context

- On mobile browsers, the top address bar and bottom navigation bar can go out of screen when you scroll down. `100vh` correspond to the height when top bar and bottom bar gets out of screen, which is larger than the height when the two bars are on screen. The modern solution is `100dvh`.
- `position: absolute` is not based on its parent. It's based on its nearest positioned ancestor (the nearest ancestor that has `position` be `relative`, `absolute` or creates stacking context).
- [Blur does not consider ambient things](https://www.joshwcomeau.com/css/backdrop-filter/#the-issue).
- If the parent's `display` is `flex` or `grid`, then the child's `float` has no effect
- If the parent's width/height is not pre-determined, then percent width/height (e.g. `width: 50%`, `height: 100%`) doesn't work. (It avoids circular dependency where parent height is determined by content height, but content height is determined by parent height.)
- `display: inline` ignores `width` `height` and `margin-top` `margin-bottom`
- Whitespace collapse. [HTML Whitespace is Broken](https://blog.dwac.dev/posts/html-whitespace/)
  - By default, newlines in html are treated as spaces. Multiple spaces together collapse into one. 
  - `<pre>` can avoid collapsing whitespace but has weird behavior in the beginning and end of content.
  - Often the spaces in the beginning and end of content are ignored, but this doesn't happen in `<a>`.
  - Any space or line break between two `display: inline-block` elements will be rendered as spacing. This doesn't happen in flexbox or grid.
- `text-align` aligns text and inline things, but doesn't align block elements (e.g. normal divs).
- By default `width` and `height` doesn't include padding and border. `width: 100%` with `padding: 10px` can still overflow the parent. `box-sizing: border-box` make the width/height include border and padding.
- [Cumulative Layout Shift](https://web.dev/articles/cls). It's recommended to specify `width` and `height` attribute in `<img>` to avoid layout shift due to image loading delay.
- File download request is not shown in Chrome dev tool because the it only shows networking in current tab, but file download is treated as in another tab. To inspect file download request, use `chrome://net-export/`.
- JS-in-HTML may interfere with HTML parsing. For example `<script>console.log('</script>')</script>` makes browser treat the first `</script>` as ending tag. [See also](https://sirre.al/2025/08/06/safe-json-in-script-tags-how-not-to-break-a-site/)

### Unicode and text encoding

- Two concepts: code point (rune), grapheme cluster:
  - Grapheme cluster is the "unit of character" in GUI.
  - For visible ascii characters, a character is a code point, a character is a grapheme cluster.
  - An emoji is a grapheme cluster, but it may consist of many code points.
  - In UTF-8, a code point can be 1, 2, 3 or 4 bytes. The byte number does not necessarily represent code point number.
  - In UTF-16, a code point can be 2 bytes or 4 bytes (surrogate pair).
  - The standard doesn't put an upper limit on how much code point can a grapheme cluster contain. But implementations usually impose a limit for performance concerns.
- Different in-memory string behaviors in different languages:
  - Rust use UTF-8 for in-memory string. `s.len()` gives byte count. Rust does not allow directly indexing on a `str`. `s.chars().count()` gives code point count. Rust is strict in UTF-8 code point validity (for example Rust doesn't allow subslice to cut on invalid code point boundary).
  - Golang string has no constraint of encoding and is similar to byte array in language level. String length and indexing works same as byte array. But the most common encoding is UTF-8. [See also](https://go.dev/blog/strings)
  - Java, C# and JS's string conceptually use UTF-16 encoding. UTF-16 works on 2-byte-units. But a code point can be 1 2-byte-unit or 2 2-byte-units (surrogate pair). Length of string is the 2-byte-unit count, not code point count. Indexing works on 2-byte-units.
  - In Python, `len(s)` gives code point count. Indexing gives a string that contains one code point.
  - In C++, `std::string` has no constraint of encoding. It can be seen as a wrapper of `std::vector<char>`. String length and indexing is based on bytes.
  - No language mentioned above do string length and indexing based on grapheme cluster.
- Some text files have byte order mark (BOM) at the beginning. For example, EF BB BF is a BOM that denotes the file is in UTF-8 encoding. It's mainly used in Windows. Some non-Windows software does not handle BOM.
- When converting binary data to string, often the invalid places are replaced by � (U+FFFD)
- [Confusable characters](https://github.com/unicode-org/icu/blob/main/icu4c/source/data/unidata/confusables.txt).
- Normalization. é can be U+00E9 or U+0065 U+0301
- [Zero-width characters](https://ptiglobal.com/the-beauty-of-unicode-zero-width-characters/), [Invisible characters](https://invisible-characters.com/)
- Line break. Windows often use CRLF `\r\n` for line break. Linux and MacOS often use LF `\n` for line break.
- [Han unification](https://en.wikipedia.org/wiki/Han_unification). Different characters in different language use the same code point. Different languages' font variants render the same character differently. Choosing the correct font variant is needed for internationalization. [HTML code](https://github.com/qouteall/qouteall-blog/blob/main/blog/2025/unicode-unification-example.html) ![](unicode_unification_example.png)



### Floating point

- NaN. Floating point NaN is not equal to any number including itself. NaN == NaN is always false (even if the bits are same). NaN != NaN is always true. Computing on NaN usually gives NaN (it can "contaminate" computation).
- There are +Inf and -Inf. They are not NaN.
- There is a negative zero -0.0 which is different to normal zero. The negative zero equals zero when using floating point comparision. Normal zero is treated as "positive zero".
- Directly compare equality may fail. Compare equality by things like `abs(a - b) < 0.00001`
- JS use floating point for all numbers. The max "safe" integer is $2^{53}-1$. The "safe" here means every integer in range can be accurately represented. Outside of the same range, some integers will become accurate. For large integer it's recommended to use `BigInt`.
  
  If a JSON contains an integer larger than that, and JS deserializes it using `JSON.parse`, the number in result will be inaccurate. The workaround is to use other ways of deserializing JSON or use string for large integer. 
  
  (Putting millisecond timestamp integer in JSON fine, as millisecond timestamp exceeds limit in year 287396. But nanosecond timestamp suffers from that issue.)
- Associativity law and distribution law doesn't strictly hold because of precision loss. Parallelizing matrix multiplication and sum dynamically using these laws can be non-deterministic. [Example](https://github.com/pytorch/pytorch/issues/75240) [Example](https://www.twosigma.com/articles/a-workaround-for-non-determinism-in-tensorflow/)
- Division is much slower than multiplication (unless using approximation). Dividing many numbers with one number can be optimized by firstly computing reciprocal then multiply by reciprocal.
- These things can make different hardware have different floating point computation results:
  - Hardware FMA (fused multiply-add) support. `fma(a, b, c) = a * b + c` (in some places `a + b * c`). Most modern hardware make intermediary result in FMA have higher precision. Some old hardware or embedded processors don't do that and treat it as normal multiply and add.
  - Floating point has a [Subnormal range](https://en.wikipedia.org/wiki/Subnormal_number) to make very-close-to-zero numbers more accurate. Most mondern hardware can handle them, but some old hardware and embedded processors treat subnormals as zero.
  - Rounding mode. The standard allows different rounding modes like round-to-nearest-ties-to-even (RNTE) or round-toward-zero (RTZ). 
    - In X86 and ARM rounding mode is thread-local mutable state can be set by special instructions. It's not recommended to touch the rounding mode as it can affect other code.
    - In GPU there is no mutable state for rounding mode. Rasterization often use RNTE rounding mode. In CUDA different rounding modes are associated by different instructions.
  - Different hardware may have different behaviors on math functions like sin, log.
  - X86 has legacy FPU which has 80-bit floating point registers and per-core rounding mode state. It's recommended to not use them.
  - ... There are many other factors that cause difference of floating point computation.
- How to increase precision in floating point computation:
  - Make computation graph shallower. For example, 3-layer computation `a * (b * (c * d))` can be less accurate than 2-layer-computation `(a * b) * (c * d)` (depends on exact case).
  - Avoid temporary result to have very large absolute value or very close-to-zero value.
  - Utilizing hardware fused operations like FMA (fused multiply-add).


### Time

- Leap second. Unix timestamp is "transparent" to leap second, which means that converting between unix timestamp and UTC time ignores leap second. The time measured in unix timestamp will stretch or squeeze near a leap second (leap smear) to hide existence of leap second.
- Time zone. UTC and Unix timestamp is globally uniform and doesn't care about time zone. But human-readable time is time-zone-dependent. It's recommended to store timestamp in database and convert to human-readable time in UI instead of storing human-readable time in database.
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
- Return in `finally` block swallows any exception thrown in the `try` or `catch` block. The method will return the value from `finally`.
- Interrupt. Some libraries ignore interrupt. Interrupt may break class initialization that contains IO.
- Thread pool does not log exception of tasks sent by `.submit()` by default. You can only get exception from the future returned by `.submit()`. Don't discard the future. And `scheduleAtFixedRate` task silently stop if exception is thrown.
- Literal number starting with 0 will be treated as octal number. (`0123` is 83)
- When debugging, debugger will call `.toString()` to local variables. Some class' `.toString()` has side effect, which cause the code to run differently under debugger. This can be disabled in IDE.

### Golang

- `append()` reuses memory region if capacity allows. Appending to a subslice can overwrite parent if they share memory region.
- `defer` executes when the function returns, not when the lexical scope exits.
- `defer` capture mutable variable.
- About `nil`:
  - There are nil slice and empty slice (the two are different). But there is no nil string, only empty string. The nil map can be read like an empty map, but nil map cannot be written.
  - Interface `nil` weird behavior. Interface pointer is a fat pointer containing type info and data pointer. If the data pointer is null but type info is not null, then it will not equal `nil`.
- Dead wait. [Understanding Real-World Concurrency Bugs in Go](https://songlh.github.io/paper/go-study.pdf)
- Different kinds of timeout. [The complete guide to Go net/http timeouts](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)


### C/C++

- Storing a pointer of an element inside `std::vector` and then grow the vector, `vector` may re-allocate content, making element pointer no longer valid.
- `std::string` created from literal string may be temporary. Taking `c_str()` from a temporary string is wrong.
- Iterator invalidation. Modifying a container when looping on it.
- `std::remove` doesn't remove but just rearrange elements. `erase` actually removes.
- Literal number starting with 0 will be treated as octal number. (`0123` is 83)
- Undefined behaviors. The compiler optimization aim to keep defined behavior the same, but can freely change undefined behavior. Relying on undefined behavior can make program break under optimization. [See also](https://russellw.github.io/undefined-behavior)
  - Accessing uninitialized memory is undefined behavior. Converting a `char*` to struct pointer can be seem as accessing uninitialized memory, because the object lifetime hasn't started. It's recommended to put the struct elsewhere and use `memcpy` to initialize it.
  - Accessing invalid memory (e.g. null pointer) is undefined behavior.
  - Integer overflow/underflow is undefined behavior. Note that unsigned integer can underflow below 0.
  - Aliasing.
    - Aliasing means multiple pointers point to the same place in memory.
    - Strict aliasing rule: If there are two pointers with type `A*` and `B*`, then compiler assumes two pointer can never equal. If they equal, it's undefined behavior. Except in two cases: 1. `A` and `B` has subtyping relation 2. converting pointer to byte pointer (`char*`, `unsigned char*` or `std::byte*`) (the reverse does not apply).
    - To do force conversion, the safe ways are `memcpy` or `std::bit_cast`
  - Unaligned memory access is undefined behavior.
- Alignment.
  - For example, 64-bit integer's address need to be disivible by 8. In ARM, accessing memory in unaligned way can cause crash.
  - Directly treating a part of byte buffer as a struct may cause alignment issue.
  - Alignment can cause padding in struct that waste space.
  - Some SIMD instructions only work with aligned data. For example, AVX instructions usually require 32-byte alignment.

### Python

- Default argument is a stored value that will not be re-created on every call.

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
- Foreign key may cause implicit locking, which may cause deadlock.
- Locking may break repeatable read isolation (it's database-specific).
- Distributed SQL database may doesn't support locking or have weird locking behaviors. It's database-specific.
- If the backend has N+1 query issue, the slowness may won't be shown in slow query log, because the backend does many small queries serially and each individual query is fast.
- Long-running transaction can cause problems (e.g. locking). It's recommended to make all transactions finish quickly.
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
  - In C#, `volatile` accesses have release-aquire ordering (CLR will use memory barrier instruction if needed)
  - `volatile` can avoid wrong optimization related to reordering and merging memory reads/writes.
- Time-of-check to time-of-use (TOCTOU).
- In SQL database, for special uniqueness constraint that doesn't fit simple unique index (e.g. unique across two tables, conditional unique, unique within time range), and constraint is enforced by application, then:
  - In MySQL (InnoDB), if in repeatable read level, application checks using `select ... for update` then insert, and the unique-checked column has index, then it works due to gap lock. (Note that gap lock may cause deadlock under high concurrency, ensure deadlock detection is on and use retrying).
  - In PostgreSQL, if in repeatable read level, application checks using `select ... for update` then insert, it's not sufficient to enforce constraint under concurrency (due to write skew). Some solutions:
    - Use serializable level
    - Don't rely on application to enforce constraint:
      - For conditional unique, use partial unique index. 
      - For uniqueness across two tables case, insert redundant data into one extra table with unique index.
      - For time range exclusiveness case, use range type and exclude constraint.
- Atomic reference counting (`Arc`, `shared_ptr`) is slow when many threads frequently change the same counter. [See also](https://pkolaczk.github.io/server-slower-than-a-laptop/)
- About read-write lock: some read-write locks doesn't support upgrading from read lock to write lock. Try to write lock when holding read lock may deadlock.

### Common in many languages

- Forget to check for null/None/nil.
- Modifying a container when for looping on it. Single-thread "data race".
- Unintended sharing of mutable data. For example in Python `[[0] * 10] * 10` does not create a proper 2D array.
- For integer `(low + high) / 2` may overflow. A safer way is `low + (high - low) / 2` (it works when `low` and `high` are non-negative)
- Short circuit. `a() || b()` will not run `b()` if `a()` returns true. `a() && b()` will not run `b()` when `a()` returns false.
- When using profiler: the profiler may by default only include CPU time which excludes waiting time. If your app spends 90% time waiting on database, the flamegraph may not include that 90% which is misleading.
- There are many different "dialects" of regular expression. Don't assume a regular expression that works in JS can work in Java.

### Linux and bash

- If the current directory is moved, `pwd` still shows the original path. `pwd -P` shows the real path.
- `cmd > file 2>&1` make both stdout and stderr go to file. But `cmd 2>&1 > file` only make stdout go to file but don't redirect stderr.
- File name is case sensitive (unlike Windows).
- There is a capability system for executables, apart from file permission sytem. Use `getcap` to see capability.
- Unset variables. If `DIR` is unset, `rm -rf $DIR/` becomes `rm -rf /`. Using `set -u` can make bash error when encountering unset variable.
- If you want a script to add variables and aliases to current shell, it should be executed by using `source script.sh`, instead of directly executing. But the effect of `source` is not permanent and doesn't apply after re-login. It can be made permanent by putting into `~/.bashrc`.
- Bash has caching between command name and file path of command. If you move one file in `$PATH` then using that command gives ENOENT. Refresh cache using `hash -r`
- Using a variable unquoted will make its line breaks treated as space.
- K8s `livenessProbe` used with debugger. Breakpoint debugger usually block the whole application, making it unable to respond health check request, so it can be killed by K8s `livenessProbe`.



### React

- Modifying state in rendering code.
- Hook used inside if or loop.
- Value not included in `useEffect` dependency array.
- Forget clean up in `useEffect`.
- Closure trap (capturing outdated state).
- Accidentally change data in wrong places (impure component).
- Forgetting `useCallback` cause unnecessary re-render.
- Passing non-memoed things into a memoed component makes memo useless.



### Git

- Rebase can rewrite history. After rebasing local branch, normal push will give weird result (because history is rewritten). Rebase should be used with force push. If remote branch's history is rewritten, pulling should use `--rebase`.
  - Force pushing with `--force-with-lease` can sometimes avoid overwriting other developers' commits. But if you fetch then doesn't pull, `--force-with-lease` cannot protect.
- Reverting a merge doesn't fully cancel the side effect of the merge. If you merge B to A and then revert, merging B to A again has no effect. One solution is to revert the revert of merge. (A cleaner way to cancel a merge, instead of reverting merge, is to backup the branch, then hard reset to commit before merge, then cherry pick commits after merge, then force push.)
- In GitHub, if you accidentally commited secret (e.g. API key) and pushed to public, even if you override it using force push, GitHub will still record that secret. [Guest Post: How I Scanned all of GitHub’s “Oops Commits” for Leaked Secrets](https://trufflesecurity.com/blog/guest-post-how-i-scanned-all-of-github-s-oops-commits-for-leaked-secrets) [Example activity tab](https://github.com/SharonBrizinov/test-oops-commit/compare/e6533c7bd729957b2eb31e88065c5158d1317c5e...9eedfa00983b7269a75d76ec5e008565c2eff2ef)
- In GitHub, if there is a private repo A and you forked it as B (also private), then when A become public, the private repo B's content is also publicly accessible, even after deleting B. [See also](https://trufflesecurity.com/blog/anyone-can-access-deleted-and-private-repo-data-github). 
- `git stash pop` does not drop the stash if there is a conflict.
- It's recommended to add `**/.DS_Store` into `.gitignore` because MacOS auto adds `.DS_Store` files into every folder.

### Networking

- Some routers and firewall silently kill idle TCP connections without telling application. Some code (like HTTP client libraries, database clients) keep a pool of TCP connections for reuse, which can be silently invalidated. To solve it you can configure system TCP keepalive.
- The result of `traceroute` is not reliable. [Traceroute isn't real](https://gekk.info/articles/traceroute.htm). Sometimes [tcptraceroute](https://linux.die.net/man/1/tcptraceroute) can be useful.
- TCP slow start can increase latency. Can be fixed by disabling `tcp_slow_start_after_idle`. [See also](https://ably.com/blog/optimizing-global-message-transit-latency-a-journey-through-tcp-configuration)
- TCP sticky packet. Nagle's algorithm delays packet sending. It will increase latency. Can be fixed by enabling `TCP_NODELAY`. [See also](https://brooker.co.za/blog/2024/05/09/nagle.html) 
- If you put your backend behind Nginx, you need to configure connection reuse, otherwise under high concurrency, connection between nginx and backend may fail, due to not having enough internal ports.
- Nginx by default buffers packet. It will delay SSE.
- The HTTP protocol does not explicitly forbit GET and DELETE requests to have body. Some places do use body in GET and DELETE requests. But many libraries and HTTP servers does not support them.
- One IP can host multiple websites, distinguished by domain name. The HTTP header `Host` and SNI in TLS handshake carries domain name, which are important. Some websites cannot be accessed via IP address.
- CORS (cross-origin resource sharing). For requests to another website (origin), the browser will prevent JS from getting response, unless the server's response contains header `Access-Control-Allow-Origin` and it matches client website. This requires configuring the backend. If you want to pass cookie to another website it involves more configuration.
  
  Generally, if your frontend and backend are in the same website (same domain name and port) then there is no CORS issue.

### Other

- YAML is space-sensitive, unlike JSON. `key:value` is wrong. `key: value` is correct.
- When using Microsoft Excel to open a CSV file, Excel will do a lot of conversions, such as date conversion (e.g. turn `1/2` and `1-2` into `2-Jan`) and Excel won't show you the original string. [The gene SEPT1 was renamed due to this Excel issue](https://en.wikipedia.org/wiki/SEPTIN1). Excel will also make large numbers inaccurate (e.g. turn `12345678901234567890` into `12345678901234500000`) and won't show you the original accurate number, because Excel internally use floating point for number.


