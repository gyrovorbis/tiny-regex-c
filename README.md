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
- Added stringifier to (sort of) go back to string form from a compiled regexp
- Broke print() function
- Broke recursive pattern matching

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
- Small code and binary size: 500 SLOC, ~3kb binary for x86. Statically #define'd memory usage / allocation.
  - NOTE: support added for user-specified storage -- Falco
- No use of dynamic memory allocation (i.e. no calls to `malloc` / `free`).
- To avoid call-stack exhaustion, iterative searching is preferred over recursive by default (can be changed with a pre-processor flag).
- No support for capturing groups or named capture: `(^P<name>group)` etc.
- Thorough testing : [exrex](https://github.com/asciimoo/exrex) is used to randomly generate test-cases from regex patterns, which are fed into the regex code for verification. Try `make test` to generate a few thousand tests cases yourself. 
- Verification-harness for [KLEE Symbolic Execution Engine](https://klee.github.io), see [formal verification.md](https://github.com/kokke/tiny-regex-c/blob/master/formal_verification.md).
- Provides character length of matches.
- Compiled for x86 using GCC 7.2.0 and optimizing for size, the binary takes up ~2-3kb code space and allocates ~0.5kb RAM :
  ```
  > gcc -Os -c re.c
  > size re.o
      text     data     bss     dec     hex filename
      2404        0     304    2708     a94 re.o
      
  ```



### API
This is the public / exported API:
```C
/* Typedef'd pointer to hide implementation details. */
typedef struct regex_t* re_t;

/* Compiles regex string pattern to a regex_t-array. */
re_t re_compile(const char* pattern);

/* Finds matches of the compiled pattern inside text. */
int  re_matchp(re_t pattern, const char* text, int* matchlength);

/* Finds matches of pattern inside text (compiles first automatically). */
int  re_match(const char* pattern, const char* text, int* matchlength);
```

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
  -  `|`        Branch Or, e.g. a|A, \w|\s
  -  `(...)`    Group

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
- Fix length with nested groups, e.g. `((ab)|b)+` =~ abbb => 7 not 4.
- Add `example.c` that demonstrates usage.
- Add `tests/test_perf.c` for performance and time measurements.
- Add optional multibyte support (e.g. UTF-8). On non-wchar systems roll our own.
- Word boundary: \b \B
- Non-greedy, lazy quantifiers (??, +?, *?, {n,m}?)
- Case-insensitive option or API. `re_matchi()`
- `re_match_capture()` with groups.
- '.' may not match '\r' nor '\n', unless a single-line option is given.

### FAQ
- *Q: What differentiates this library from other C regex implementations?*

  A: Well, the small size for one. 500 lines of C-code compiling to 2-3kb ROM, using very little RAM.

### License
All material in this repository is in the public domain.

