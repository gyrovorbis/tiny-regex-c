Falco Girgis's Changelist:
- merged several bugfixes that were just sitting as pending PRs
- merged several features that were just sitting as pending PRs
- added CMake support
- exposed configuration definitions as CMake options
- modified internal symbol representation to be far more compact
  - everything is byte-packed as tightly as humanly possible
  - character class substrings are now stored within the symbol 
  - far more cache coherent
- added support for user-supplied compiled pattern storage
  - allows you to heap allocate and maintain more than one compiled pattern
- Fixed multi matches `{n}`, `{,m}`, `{n,}`, `{n, m}`, which were only partially working previously
  - a `{n}` that wasn't the first element advanced the text by the cumulative match length
  - a `{n}` ending at end-of-text returned success without matching the rest of the pattern
- Implemented real alternation: `|` separates whole alternatives, tried left to right
  - previously `hello|world` behaved as `hell(o|world)`, so only single-character alternatives worked
  - works at top level and inside groups; `^` and `$` bind per-alternative
- Fixed quantifiers inside groups, which previously hung the matcher (`(ab?)c` on `ac`)
  - matching now continues through `)` into the rest of the pattern, so `?`/`*`/`+` inside a group backtrack against what follows it
- Fixed `(ab)c` matching `ab` (group ending at end-of-text returned success early)
- Added stringifier to (sort of) go back to string form from a compiled regexp
- Broke print() function (and with it the upstream `make test` harness)

#
![CI](https://github.com/kokke/tiny-regex-c/workflows/CI/badge.svg)
# tiny-regex-c
# A small regex implementation in C
### Description
Small and portable [Regular Expression](https://en.wikipedia.org/wiki/Regular_expression) (regex) library written in C. 

Design is inspired by Rob Pike's regex-code for the book *"Beautiful Code"* [available online here](http://www.cs.princeton.edu/courses/archive/spr09/cos333/beautiful.html).

Supports a subset of the syntax and semantics of the Python standard library implementation (the `re`-module).

**I will gladly accept patches correcting bugs.**

### Design goals
The main design goal of this library is to be small, correct, self contained and use few resources while retaining acceptable performance and feature completeness. Clarity of the code is also highly valued.

### Notable features and omissions
- Small code and binary size: ~1100 lines including comments, ~5.5kb binary for x86. Statically #define'd memory usage / allocation.
  - NOTE: support added for user-specified storage -- Falco
- No use of dynamic memory allocation (i.e. no calls to `malloc` / `free`).
- Matching is recursive; nesting depth is bounded by the number of quantifiers and groups in the pattern, not by the text length.
- No support for capturing groups or named capture: `(^P<name>group)` etc.
- Quantifiers (`?`, `*`, `+`, `{n}`...) apply to the single preceding atom or character class only. They cannot be applied to a group: `(ab)+` does not match anything.
- Upstream's [exrex](https://github.com/asciimoo/exrex)-based random test harness (`make test`) and [KLEE](https://klee.github.io) verification harness (see [formal verification.md](https://github.com/kokke/tiny-regex-c/blob/master/formal_verification.md)) are present but no longer build against this fork. Coverage lives in libGimbal's `GblPatternTestSuite`.
- Provides character length of matches.
- Compiled for x86 with GCC 16 and optimizing for size:
  ```
  > gcc -Os -c re.c
  > size re.o
      text     data     bss     dec     hex filename
      5466        0     180    5646    160e re.o
  ```



### API
This is the public / exported API:
```C
/* Typedef'd pointer to hide implementation details. */
typedef struct regex_t* re_t;

/* Compiles regex string pattern into the internal static buffer. */
re_t re_compile(const char* pattern);

/* Compiles regex string pattern into a caller-supplied buffer, returning the number of bytes used. */
re_t re_compile_to(const char* pattern, unsigned char* re_data, unsigned* bytes);

/* Returns the size in bytes of a compiled pattern. */
unsigned re_size(re_t pattern);

/* Compares two compiled patterns for equality. */
int  re_compare(re_t pattern1, re_t pattern2);

/* Reconstructs (approximately) a regex string from a compiled pattern. */
void re_string(re_t pattern, char* buffer, unsigned* size);

/* Finds matches of the compiled pattern inside text. */
int  re_matchp(re_t pattern, const char* text, int* matchlength);

/* Finds matches of pattern inside text (compiles first automatically). */
int  re_match(const char* pattern, const char* text, int* matchlength);
```

`re_compile()` uses one internal buffer, so only the most recently compiled pattern is valid; use `re_compile_to()` to keep several compiled patterns alive at once.

### Configuration
- `RE_DOT_MATCHES_NEWLINE` (default `1`): whether `.` matches `\r` and `\n`.
- `MAX_REGEXP_OBJECTS` (default `30`): maximum number of symbols in a pattern compiled with `re_compile()`.
- `MAX_CHAR_CLASS_LEN` (default `40`): maximum total length of character-class contents in a pattern.

### Supported regex-operators
The following features / regex-operators are supported by this library.


  -  `.`         Dot, matches any character
  -  `^`         Start anchor, matches beginning of string
  -  `$`         End anchor, matches end of string
  -  `*`         Asterisk, match zero or more (greedy)
  -  `+`         Plus, match one or more (greedy)
  -  `?`         Question, match zero or one (non-greedy)
  -  `{n}`       Exact Quantifier
  -  `{n,}`      Match n or more times
  -  `{,m}`      Match m or less times
  -  `{n,m}`     Match n to m times
  -  `[abc]`     Character class, match if one of {'a', 'b', 'c'}
  -  `[^abc]`   Inverted class, match if NOT one of {'a', 'b', 'c'}
  -  `[a-zA-Z]` Character ranges, the character set of the ranges { a-z | A-Z }
  -  `\s`       Whitespace, '\t' '\f' '\r' '\n' '\v' and spaces
  -  `\S`       Non-whitespace
  -  `\w`       Alphanumeric, [a-zA-Z0-9_]
  -  `\W`       Non-alphanumeric
  -  `\d`       Digits, [0-9]
  -  `\D`       Non-digits
  -  `\xXX`     Hex-encoded byte
  -  `|`        Alternation, e.g. `cat|dog`, `(ab|cd)e`. Alternatives are tried left to right; the first that matches wins.
  -  `(...)`    Group. Non-capturing, and cannot be quantified (see above).

### Usage
Compile a regex from ASCII-string (char-array) to a custom pattern structure using `re_compile()`.

Search a text-string for a regex and get an index into the string, using `re_match()` or `re_matchp()`.

The returned index points to the first place in the string, where the regex pattern matches.

The integer pointer passed will hold the length of the match.

If the regular expression doesn't match, the matching function returns an index of -1 to indicate failure.

### Examples
Example of usage:
```C
/* Standard int to hold length of match */
int match_length;

/* Standard null-terminated C-string to search: */
const char* string_to_search = "ahem.. 'hello world !' ..";

/* Compile a simple regular expression using character classes, meta-char and greedy quantifiers: */
re_t pattern = re_compile("[Hh]ello [Ww]orld\\s*[!]?");

/* Check if the regex matches the text: */
int match_idx = re_matchp(pattern, string_to_search, &match_length);
if (match_idx != -1)
{
  printf("match at idx %i, %i chars long.\n", match_idx, match_length);
}
```

For more usage examples I encourage you to look at the code in the `tests`-folder.

### TODO
- Quantifiers on groups, e.g. `(ab)+`, `((ab)|b)+`.
- Restore `re_print()` and the upstream test harness.
- Add `example.c` that demonstrates usage.
- Add `tests/test_perf.c` for performance and time measurements.
- Add optional multibyte support (e.g. UTF-8). On non-wchar systems roll our own.
- Word boundary: \b \B
- Non-greedy, lazy quantifiers (??, +?, *?, {n,m}?)
- Case-insensitive option or API. `re_matchi()`
- `re_match_capture()` with groups.
- '.' may not match '\r' nor '\n', unless a single-line option is given (see `RE_DOT_MATCHES_NEWLINE`).

### FAQ
- *Q: What differentiates this library from other C regex implementations?*

  A: Well, the small size for one. About 1100 lines of C compiling to ~5kb ROM, using very little RAM.

### License
All material in this repository is in the public domain.

