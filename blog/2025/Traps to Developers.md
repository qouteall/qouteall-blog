---
date: 2025-08-03
tags:
  - Programming
---
# Traps to Developers

A lot of bugs come from developer not knowing the trap in the tool they use. Here are the summarization of some traps I know:

<!-- truncate -->

## Summarazation of traps

### Java

- `==` compares object reference, not content. 
- Forget to override `equals` and `hashcode`. It will use object identity equality by default in map key and set.
- Mutate the content of map key object (or set element object) makes the container malfunciton.
- A method that returns `List<T>` may sometimes return mutable `ArrayList`, but sometimes return `Collections.emptyList()` which is immutable. Trying to modify on `Collections.emptyList()` throws `UnsupportedOperationException`.
- A method that returns `Optional<T>` may return `null`.
- Return in `finally` block swallows any exception thrown in the `try` or `catch` block. The method will return the value from `finally`.
- Interrupt. Some libraries ignore interrupt. Interrupt may break class initialization that contains IO.
- Thread pool does not log exception of tasks by default. You can only get exception from the future object. Don't discard the future. And `scheduleAtFixedRate` task silently stop if exception is thrown.

### Golang

- `append()` may reuse underlying array if capacity allows. Appending to a subslice can overwrite parent if they share memory region. `s[:len(s):len(s)]` creates a new slice with capacity equal to its length, preventing this.
- `defer` executes when the function returns, not when the lexical scope exits.
- `defer` capture mutable variable.
- There are nil slice and empty slice (the two are different). But there is no nil string, only empty string.
- Json deserialization field name is case-insensitive.
- Interface `nil` weird behavior. Interface pointer is a fat pointer containing type info and data pointer. If the data pointer is null but type info is not null, then it will not equal `nil`.
- Dead wait. [Understanding Real-World Concurrency Bugs in Go](https://songlh.github.io/paper/go-study.pdf)


### C/C++

- Storing a pointer of an element inside `std::vector` and then grow the vector, `vector` may re-allocate content and the previous element pointer may no longer be valid.
- All `std::string` created from literal string are temporary. Taking `c_str()` from a temporary string is wrong.
- Iterator invalidation. Modifying a container when looping on it.
- `std::remove` doesn't remove but just rearrange elements. `erase` actually removes.
- Unsigned integers can underflow.

### Python

- Default argument is a stored value that will not be re-created on every call.
- Numpy and Pytorch has different transpose.

### Common in many languages

- Forge to check for null.
- Multithreading data race.
- Modifying a container when for looping on it. Single-thread "data race".
- Unintended sharing of mutable data. For example in Python `[[0] * 10] * 10` does not create a proper 2D array.
- A number starting with 0 will be seen as octonary number (`0123` is 83).
- For integer `(low + high) / 2` may overflow. A safer way is `low + (high - low) / 2`
- When using profiler: the profiler may by default only include CPU time which excludes waiting time. If your app spends 90% time waiting on database, the flamegraph may not include that 90% which is misleading.

### Floating point

- NaN. Floating point NaN is not equal to any number including itself. NaN always give false in comarision. Computing on NaN usually gives NaN.
- There are +Inf and -Inf. They are not NaN. 
- There is a negative zero -0 which is different to normal zero. The negative zero equals zero when using floating point comparision.
- Directly compare equality may fail. Compare equality by things like `abs(a - b) < 0.0001`
- JS use floating point for all numbers. The max safe integer is $2^{53}-1$ if not using BigInteger. If a JSON contains an integer larger than that, and JS deserializes it using `JSON.parse`, the number in result will be inaccurate. The workaround is to use other ways of deserializing JSON or use string for large integer.
- Avoid accumulating error. For example, if you rotate 1 degree per frame: don't multiply one 1-degree-rotation matrix cumulatively per frame. Compute angle from time and then compute rotation matrix every frame.
- Associativity law and distribution law doesn't strictly hold because of inaccuracy. Parallelizing matrix multiplication and sum can be non-deterministic. https://github.com/pytorch/pytorch/issues/75240 https://www.twosigma.com/articles/a-workaround-for-non-determinism-in-tensorflow/
- Division is usually much slower than multiplication.


### SQL Databases

- Null is special. `x = null` doesn't work. `x is null` works. Unique index allows duplicating null. But `select distinct` may treat nulls as the same (this is database-specific).
- Date implicit conversion can be timezone-dependent.
- Complex join with disctinct may be slower than nested query. [Don't use DISTINCT as a "join-fixer"](https://www.red-gate.com/simple-talk/databases/sql-server/t-sql-programming-sql-server/dont-use-distinct-as-a-join-fixer/)
- In MySQL, if string field doesn't have `character set utf8mb4` then it will error if you try to insert a text containing 4-byte UTF-8 code point.
- MySQL default to case-insensitive.
- MySQL can do implicit conversion by default. `SELECT '123abc' + 1;` gives 124.
- MySQL gap lock may cause deadlock.
- In MySQL you can select a field and group by another field. It gives nondeterministic result.
- In SQLite the field type doesn't matter unless the table is `strict`.
- Foreign key may cause implicit locking, which may cause deadlock.
- Locking may break repeatable read isolation (it's database-specific).
- Distributed SQL database may doesn't support locking or have weird locking behaviors. It's database-specific.
- If the backend has N+1 query issue, the slowness may won't be shown in slow query log, because the backend does many small queries serially and each individual query is fast.



### Linux and bash

- If the current directory is moved, `pwd` still shows the original path. `pwd -P` shows the real path.
- `cmd > file 2>&1` make both stdout and stderr go to file. But `cmd 2>&1 > file` only make stdout go to file but don't redirect stderr.
- File name is case sensitive (unlike Windows).
- An alternative file permission system. `getcap`
- Unset variables. If `DIR` is unset, `rm -rf $DIR/` becomes `rm -rf /`
- If you want a script to add variables and aliases to current shell, it should be executed by using `source script.sh`, instead of directly executing.
- Bash has caching between command name and file path of command. If you move one file in `$PATH` then using that command gives ENOENT. Refresh cache using `hash -r`



### React

- Modifying state in rendering code.
- Hook used inside if or loop.
- Value not included in `useEffect` dependency array.
- Forget clean up in `useEffect`.
- Closure trap (capturing outdated state).
- Accidentally change data in wrong places (impure component).
- Render code read value inside ref.
- Forgetting `useCallback` cause unnecessary re-render.
- Passing non-memoed things into a memoed component makes memo useless.



### HTML and CSS

- Whitespace collapse. [HTML Whitespace is Broken](https://blog.dwac.dev/posts/html-whitespace/)
- `position: absolute` is not based on its parent. It's based on its nearest positioned ancestor (the nearest parent that has `position` be `relative`, `absolute`, `fixed` or `sticky` or has `transform`).
- Normally `position: fixed` positions relative to viewport. However, if its parent (or ancestor) has `transform` then its position is relative to transformed ancestor, not viewport. The same happens when one of its ancestor has `perspective` or `filter`.
- If you don't specify `min-with`, it will be `auto`, and min width will be determined by content. It has higher priority than many other CSS attributes. With it, `flex-shrink` may not work, `overflow: hidden` may not work, `width: 0` may not work, `max-width: 100%` may not work. It's recommended to set `min-width: 0`.
- `width: auto` tries fill available space in parent. But `height: auto` tries to just expand to fit content.
- Horizontal margin is different to vertical margin. `margin: 0 auto` centers horizontally. But `margin: auto 0` normally become `margin: 0 0` (in a flexbox with `flex-direction: column`, `margin: auto 0` can center vertically).
- Margin collapse. One way to avoid margin collapse is `display: flow-root` which creates a BFC (Block formatting context).
- On mobile Safari, the top address bar and bottom navigation bar can go out of screen when you scroll down. `100vh` correspond to the height when top bar and bottom bar gets out of screen, which is larger than the height when the two bars are on screen. The solution is `100dvh` (may not be supported in old browsers).
- [Blur issue](https://www.joshwcomeau.com/css/backdrop-filter/#the-issue)
- TODO `transform` has side effects on `z-index`, `position: absolute`, `overflow: visible`, `background-attachment: fixed`, subpixel antialiasing, `%` values, `position: sticky`, hit testing.
- TODO `border` `padding` `overflow` `display: table` `display: inline-block` can affect `margin`
- TODO `position: absolute` affects `float`

### Git

- Rebase rewrites history. Rebase should be used with force push (because it rewrites history). If a branch was force pushed, pulling should use rebase.
- Reverting a merge doesn't cancel the effect of the merge. If you merge B to A and then revert, merging B to A again has no effect.
- In GitHub, if you accidentally commited secret (e.g. API key) and pushed to public, even if you override it using force push, GitHub will still record that secret. [Guest Post: How I Scanned all of GitHub’s “Oops Commits” for Leaked Secrets](https://trufflesecurity.com/blog/guest-post-how-i-scanned-all-of-github-s-oops-commits-for-leaked-secrets) [Example activity tab](https://github.com/SharonBrizinov/test-oops-commit/compare/e6533c7bd729957b2eb31e88065c5158d1317c5e...9eedfa00983b7269a75d76ec5e008565c2eff2ef)

### Networking

- Some routers and firewall silently kill idle TCP connections without telling application. Some code (like HTTP client libraries, database clients) keep a pool of TCP connections for reuse, which can be silently invalidated. To solve it you can configure system TCP keepalive configuration. For HTTP you can use `Connection: keep-alive` `Keep-Alive: timeout=30, max=1000` header.
- The result of `traceroute` is not reliable. [Traceroute isn't real](https://gekk.info/articles/traceroute.htm). Sometimes [tcptraceroute](https://linux.die.net/man/1/tcptraceroute) can be useful.
- The `tcp_slow_start_after_idle` setting, if enabled, can increase latency. [Optimizing global message transit latency: a journey through TCP configuration](https://ably.com/blog/optimizing-global-message-transit-latency-a-journey-through-tcp-configuration)
- If `TCP_NODELAY` is not enabled, latency will increase. [It’s always TCP_NODELAY. Every damn time.](https://brooker.co.za/blog/2024/05/09/nagle.html) 
- If you put your backend behind nginx, you need to configure connection reuse, otherwise when there are many concurrent requests the connection between nginx and backend may fail due to not having enough internal ports.
- Nginx by default buffers packet. It will delay SSE.
- The HTTP protocol does not explicitly forbit GET and DELETE requests to have body. Some places do use body in GET and DELETE requests. But many libraries and HTTP servers does not support them.
- One IP can host multiple websites, distinguished by domain name. The http header `Host` and SNI in TLS handshake is important in distinguishing. You cannot make request to some websites using IP address.
- CORS (cross-origin resource sharing). When sending HTTP(S) request to another website (origin), the browser will stop JS to get response unless the server's response contains header `Access-Control-Allow-Origin` and it matches client website. This requires configuring the backend. If you want to pass cookie to another website it involves more configuration. CORS is complex and has many details.
  
  Generally, if your frontend and backend are in the same website (same domain name and port) then there is no CORS issue.

### Unicode and text encoding

- It's recommended to use UTF-8 for all files and communications to avoid troubles related to encoding.
- Two concepts: code point (rune), grapheme cluster:
  - Grapheme cluster is the "unit of character" in GUI.
  - For ascii characters, a character is a code point, a character is a grapheme cluster.
  - An emoji is a grapheme cluster, but it may consist of many code points.
  - In UTF-8, a code point can be 1 byte, 2 bytes or 4 bytes. The byte number does not necessarily represent code point number.
  - In UTF-16, a code point can be 2 bytes or 4 bytes (surrogate pair).
  - The standard doesn't put an upper limit on how much code point can a grapheme cluster contain. But implementations usually impose a limit for performance concerns.
- Different in-memory string behaviors in different languages:
  - Rust use UTF-8 for in-memory string. `s.len()` gives byte count. Rust does not allow directly indexing on a `str`. `s.chars().count()` gives code point count. Rust is strict in UTF-8 validity (for example Rust doesn't allow subslice to cut on invalid code point boundary).
  - Golang use UTF-8 for in-memory string. `len(s)` gives byte count. `s[i]` works same as byte array. `utf8.RuneCountInString(s)` gives code point count.
  - Java, C#, JS use UTF-16-like encoding for in-memory string. UTF-16 works on 2-byte-units. But a code point can be 1 2-byte-unit or 2 2-byte-units (surrogate pair). Length of string is the unit count, not code point count. Indexing works on 2-byte-units.
  - In Python, `len(s)` gives code point count. Indexing gives a string that contains one code point.
  - No language mentioned above do string length and indexing based on grapheme cluster.
- Some text files have byte order mark (BOM) at the beginning. For example, EF BB BF is a BOM that denotes the file is in UTF-8 encoding. It's mainly used in Windows. Some non-Windows software does not handle BOM.
- [Confusable characters](https://github.com/unicode-org/icu/blob/main/icu4c/source/data/unidata/confusables.txt).
- Normalization. é can be U+00E9 or U+0065 U+0301
- [Zero-width characters](https://ptiglobal.com/the-beauty-of-unicode-zero-width-characters/), [Invisible characters](https://invisible-characters.com/)


### Time

- Leap second. Unix timestamp is "transparent" to leap second, which means that converting between unix timestamp and UTC time ignores the existence of leap second. The time measured in unix timestamp will stretch or squeeze near a leap second.
- Time zone. UTC and Unix timestamp is globally uniform and doesn't care about time zone. But human-readable time is time-zone-dependent. It's recommended to store timestamp in database and convert to human-readable time in UI instead of storing human-readable time in database.
- Daylight Saving Time (DST): In some region people adjust clock forward by one hour in warm seasons.
- Time may "go backward" due to NTP sync.
- It's recommended to configure the server's time zone as UTC. Different nodes having different time zones will cause trouble in distributed system. After changing system time zone, the database may need to be reconfigured or restarted.


### Other

- YAML is space-sensitive, unlike JSON. `key:value` is wrong. `key: value` is correct.


