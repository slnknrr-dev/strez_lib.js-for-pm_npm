# @slnknrr/str-ez
> Only the README.md is here. I publish the code on Codeberg. [view →](https://codeberg.org/slnknrr/strez_lib.js-for-pm_npm).

**The friendly face of [`@slnknrr/str-im`](https://www.npmjs.com/package/@slnknrr/str-im): the same zero-allocation string engine behind familiar names, plain booleans and ready arrays.**

> Strings giving you *str-ess*? Take [**str-ez**](https://www.npmjs.com/package/@slnknrr/str-ez). Want them to *str-eam*? Take [**str-im**](https://www.npmjs.com/package/@slnknrr/str-im). Same engine, two accents: `str-ez` says `startsWith` and returns `true`; `str-im` says `starts` and returns `-1` because it has more to tell you.

`str-im` is precise and terse: `ulen`, `bcut`, `consc`, predicates that return `0 | 1 | -1`, lazy iterators everywhere, a `NotFound` exception for a missing substring. That is the right contract for an engine. It is not the contract you want in application code, where a string utility should read like `startsWith`, return `true`, and give you an array you can `map` over.

`str-ez` is that layer — and nothing more. Every function is one call into the engine plus a translation at the boundary. No logic of its own, no state, no configuration.

```js
import { byteLength, truncateBytes, uniqueChars, isMixedScript, isSimilar, lines, mutable } from '@slnknrr/str-ez';

byteLength('héllo 👾');                 // 11  — utf-8 bytes, computed without encoding
truncateBytes('hi 👾!', 7);             // 'hi 👾' — never half an emoji
uniqueChars('mississippi');            // ['m', 'i', 's', 'p']
uniqueChars('mississippi', 2);         // ['m', 'i'] — stops reading at the 2nd distinct one
isMixedScript('раypal');               // true — Latin + Cyrillic look-alikes
isSimilar('kitten', 'sitting', 2);     // false — more than 2 edits, decided without the full matrix
lines('a\r\nb\nc');                     // ['a', 'b', 'c']
mutable('  Héllo,,  wörld!!  ')         // one buffer, edited in place, one string at the end
  .trim().squeeze(',! ').remove('l').foldCase().toString(); // 'héo, wörd!'
```

---

## What changes at the boundary

Five rules. Everything else is the engine's behavior, unchanged.

| | `str-im` | `str-ez` |
|---|---|---|
| Predicates | `0` no · `1` yes · `-1` matched in full | `boolean` — `-1` is a yes |
| Iterators | lazy `IterableIterator`, `max` argument | `Array`, same-position `limit` argument keeps the early exit |
| Missing target | throws `NotFound` | slice functions return `''` |
| Saturated counts | `Infinity` = "at least this many" | `Infinity` kept; boolean companions (`hasRun`, `hasDistinct`, `isSimilar`) answer the yes/no form |
| `TypeError` / `RangeError` | thrown synchronously, at the call | unchanged — the caller is wrong, and hiding it is the opposite of easy |

Inputs are never mutated — except by [`mutable()`](#15--mutable-string--mutable), whose whole point is to mutate one buffer in place. Where the engine returns the source by reference when nothing changed (`remove`, `squeeze`, `trim`, `truncate*`), so does `str-ez`.

## What it is NOT

- **Not a second engine.** Every character-level algorithm lives in `str-im`. If you need to configure one (`_strim(overrides)`, custom sources, alphabets, confusables), import `str-im` directly; `str-ez` always uses the default instance.
- **Not lazy.** Arrays are materialized. If you want to pull one element and stop, use the engine's iterators — `str-ez` keeps `limit` so the common "first *n*" case still stops early.
- **Not a lodash replacement.** It adds what `String` lacks: utf-8 bytes, graphemes, display width, distinct characters, runs, scripts, homoglyphs, edit distance, multi-pattern search, quote-aware split. It does not re-implement `padStart` or `capitalize`.

---

## Install

```sh
npm install @slnknrr/str-ez
```

```js
import ez from '@slnknrr/str-ez';                          // one frozen object with every function
import { startsWith, uniqueChars } from '@slnknrr/str-ez'; // or named imports — tree-shakeable
```

Requirements: **Node ≥ 20**, ESM only. `@slnknrr/str-im` ≥ 1.1.0 is the single dependency and is installed with the package.

---

## API

97 functions and one class: 75 wrappers, one per engine method, 22 compositions built from them (marked **+**), and [`MutableString`](#15--mutable-string--mutable). The `str-im` column names the engine method behind each wrapper.

`str` is a string. `src` is any engine source: a `string`, a utf-8 `Uint8Array`/`Buffer`, or an `Array<number>` of code points. `target` is a string to locate or a number treated as a position. `needles` is one string, one code point, or an iterable of them — many needles are searched in a single pass. `limit` is optional and stops reading once that many results are in hand.

### 1 · Slices relative to a target — `''` when the target is absent
| Function | str-im | Returns |
|---|---|---|
| `after(str, target, length?)` | `substr` | the piece after the first `target`, at most `length` code units |
| `afterLast(str, target, length?)` | `lsubstr` | after the last |
| `before(str, target, length?)` | `substrb` | before the first |
| `beforeLast(str, target, length?)` | `lsubstrb` | before the last |
| **+** `between(str, open, close)` | | between the first `open` and the next `close` |

### 2 · Predicates — `boolean`
| Function | str-im |
|---|---|
| `startsWith(str, target)` | `starts` |
| `endsWith(str, target)` | `ends` |
| `includes(str, target)` | `contains` |

A numeric `target` compares length, as in the engine: `startsWith('abc', 2)` is "at least 2 code units long".

### 3 · Counters — `number`, accept any `src`
| Function | str-im | Counts |
|---|---|---|
| `charCount(src)` | `len` | characters (code points) |
| `astralCount(src)` | `wlen` | code points ≥ U+10000 (surrogate pairs in utf-16) |
| `uniqueCount(src)` | `ulen` | distinct characters |
| `byteLength(src)` | `blen` | **utf-8 bytes, without encoding** |
| `graphemeCount(src)` | `glen` | grapheme clusters (UAX #29) |
| `displayWidth(src)` | `dlen` | terminal columns |

### 4 · Dropping substrings
| Function | str-im | Returns |
|---|---|---|
| `remove(str, needles)` | `clear` | a new string without the needles (same string if none matched) |
| `removedIndices(str, needles)` | `omitn` | `number[]` — where the dropped characters were |
| `keptChars(str, needles)` | `omit` | `string[]` — what remains, one character per element |

### 5 · Unique characters — `string[]`, first-seen order, optional `limit`
| Function | str-im | Yields each unique… |
|---|---|---|
| `uniqueChars(str, limit?)` | `wchar` | character (code point) — an emoji is one element |
| `uniqueCodeUnits(str, limit?)` | `char` | utf-16 code unit — a surrogate pair is two |
| `uniqueCodeUnitsFromEnd(str, limit?)` | `rchar` | code unit, read from the end |
| `uniqueCharsIgnoreCase(str, limit?)` | `ichar` | character, case-folded (`A` ≡ `a`) |
| `uniqueVowels` / `uniqueConsonants` | `vchar` / `cchar` | vowel / consonant (engine alphabet: Latin + Cyrillic) |
| `uniqueUppercase` / `uniqueLowercase` | `uchar` / `lchar` | `\p{Lu}` / `\p{Ll}` |
| `uniqueDigits` / `uniquePunctuation` / `uniqueSymbols` | `nchar` / `pchar` / `schar` | `\p{Nd}` / `\p{P}` / `\p{S}` |
| `uniqueMarks` / `uniqueInvisibles` | `mchar` / `zchar` | `\p{M}` / `\p{Cc}` ∪ `\p{Cf}` |
| `uniqueGraphemes(str, limit?)` | `gchar` | grapheme cluster |
| **+** `hasDistinct(str, n)` → `boolean` | | at least `n` distinct characters? Stops at the n-th |

### 6 · Runs — consecutive repeats
| Function | str-im | Returns |
|---|---|---|
| `leadingRun(str, limit?)` | `consc` | length of the first run; `Infinity` once `limit` is reached |
| `trailingRun(str, limit?)` | `rconsc` | length of the last run |
| `runChars(str, limit?)` / `runCharsFromEnd` | `cons` / `rcons` | `string[]` — the character of each run |
| `runLengths(str, limit?)` / `runLengthsFromEnd` | `consg` / `rconsg` | `number[]` — the length of each run |
| `uniqueRunChars` / `uniqueRunCharsFromEnd` | `ucons` / `urcons` | run characters, skipping those seen in an earlier run |
| `squeeze(str, chars?, keep = 1)` | `flat` | collapse repeats: `'привееет'` → `'привет'` |
| **+** `runs(str)` | | `{ char, length }[]` — run-length encoding |
| **+** `hasRun(str, n)` → `boolean` | | any run of at least `n`? Stops at the first |

### 7 · Access by index — negative counts from the end, out of range → `undefined`
| Function | str-im | Indexed by |
|---|---|---|
| `charAt(str, index)` | `whas` | code points: `charAt('a😀b', 1)` → `'😀'` |
| `unitAt(str, index)` | `has` | utf-16 code units, like `str.at(i)` |

### 8 · Bytes and boundaries
| Function | str-im | Returns |
|---|---|---|
| `invalidUtf8At(buf)` | `badn` | index of the first invalid utf-8 byte, or `-1` |
| `bomLength(src)` | `bom` | byte length of a leading BOM, or `0` |
| `truncate(src, max, ellipsis?)` | `cut` | at most `max` code points, never splitting a pair |
| `truncateBytes(src, max, ellipsis?)` | `bcut` | at most `max` **utf-8 bytes**, never splitting a character |
| `truncateGraphemes(src, max, ellipsis?)` | `wcut` | at most `max` grapheme clusters |
| **+** `isValidUtf8(buf)` / `hasBom(src)` | | `boolean` |
| **+** `fitsChars` / `fitsBytes` / `fitsGraphemes` `(src, max)` | | `boolean` — reads no further than `max` |

`buf` is a `Uint8Array`; `bomLength` / `hasBom` take a string or a `Uint8Array`. `truncate*` return the source's own type — a string, a `Uint8Array`, or an array — and the source itself when it already fits. `ellipsis` is charged against the same budget: `truncate('hello world', 5, '…')` → `'hell…'`.

### 9 · Search and compare — indices in code units
| Function | str-im | Returns |
|---|---|---|
| `indexOfAll(str, needles, { overlapping }?)` | `find` | `number[]` — every match, left to right |
| `lastIndexOfAll(str, needles, opts?)` | `lfind` | every match, right to left |
| `indexOfAllIgnoreCase(str, needles, opts?)` | `ifind` | every match, case-folded |
| `distance(a, b, max?, { damerau }?)` | `near` | edit distance, or `Infinity` if it exceeds `max` |
| `commonPrefixLength(a, b)` / `commonSuffixLength` | `com` / `comb` | length in code points; `b` may be many strings |
| **+** `indexOfAny(str, needles)` / `lastIndexOfAny` | | first / last match, or `-1`; stops at the first |
| **+** `includesAny(str, needles)` | | `boolean` |
| **+** `countOf(str, needles)` | | number of non-overlapping matches |
| **+** `isSimilar(a, b, max, opts?)` | | within `max` edits? `max` is required |
| **+** `commonPrefix(a, b)` / `commonSuffix(a, b)` | | the shared text itself |

Matches are non-overlapping and leftmost-longest by default; `{ overlapping: true }` reports every match. With `max`, `distance` does `O(n·max)` work instead of `O(n·m)`.

### 10 · Segmentation and parsing
| Function | str-im | Returns |
|---|---|---|
| `split(str, sep?, opts?)` | `split` | `string[]`; `sep` absent ⇒ line breaks (`\r\n` \| `\n` \| `\r` as one) |
| `splitRanges(str, sep?, opts?)` | `splitn` | `[start, end][]` — boundaries instead of pieces |
| `closingBracket(str, i, opts?)` | `pair` | index of the bracket closing the one at `i`, or `-1` |
| `openingBracket(str, i, opts?)` | `pairb` | index of the bracket opening the one at `i`, or `-1` |
| `indentation(str)` | `ind` | `{ char, size, min, mixed }` |
| `dedent(str)` | `dedent` | the common indent stripped |
| `escape(str, chars?, opts?)` / `unescape(str, opts?)` | `esc` / `unesc` | escape / unescape a character set (`\` by default) |
| `trim` / `trimStart` / `trimEnd` `(str, chars?)` | `trim` / `ltrim` / `rtrim` | trim a custom set; Unicode `White_Space` by default |
| **+** `lines(str)` | | `split(str)` on line breaks |

`opts` for `split` / `splitRanges`: `{ limit, quote, quotes, esc, nest }`. For the bracket functions: `{ pairs, quote, quotes, esc, nest }`, default pairs `()` `[]` `{}` `<>`. Quote-aware mode never splits or matches inside quotes — CSV and shell-style input.

### 11 · Unicode analysis
| Function | str-im | Returns |
|---|---|---|
| `scripts(str, limit?)` | `scr` | `string[]` — script names in first-seen order |
| `isMixedScript(str)` | `mix` | `boolean`, ignoring Common / Inherited; no script at all is not mixed |
| `skeleton(str)` | `skel` | TR39-style skeleton — visually equal strings collapse |
| `deburr(str)` | `deburr` | diacritics stripped (search and sorting, not display) |
| **+** `looksLike(a, b)` | | same skeleton? `looksLike('pаypal', 'paypal')` → `true` |

### 12 · Metrics
| Function | str-im | Returns |
|---|---|---|
| `entropy(str, base = 2, n = 1)` | `ent` | Shannon entropy; bits by default, over n-grams for `n > 1` |
| `ngramIndices(str, n = 2)` | `wgram` | `number[]` — window starts, windows over code points |
| `ngramUnitIndices(str, n = 2)` | `gram` | window starts, windows over code units |
| `rollingHashes(str, n = 2)` | `hash` | Rabin–Karp hash of each window |
| **+** `ngrams(str, n = 2)` | | `string[]` — the n-grams themselves |

### 13 · Second tier
| Function | str-im | Returns |
|---|---|---|
| `matchesGlob(str, pattern, { nocase }?)` | `glob` | `* ? [abc] [a-z] [!neg]`, no regexp |
| `namingStyle(str)` | `style` | `'camel' \| 'pascal' \| 'snake' \| 'screaming' \| 'kebab' \| 'dot' \| 'mixed' \| 'lower' \| 'unknown'` |
| `wrapLines(str, width)` | `wrap` | `string[]` — wrapped by display width |
| **+** `wrap(str, width)` | | the same, joined with `\n` |

### 14 · Everyday questions
| Function | Built on | Returns |
|---|---|---|
| **+** `isBlank(str)` | `trim` | empty or Unicode `White_Space` only |
| **+** `isAscii(str)` | `blen` | every character below U+0080 — one pass, no strings built |

### 15 · Mutable string — `mutable`
| Function | str-im | Returns |
|---|---|---|
| `mutable(src)` / `new MutableString(src)` | `mut` | a `MutableString` over a `string` or a utf-8 `Uint8Array` |

Everything else in this package returns a new value. `mutable()` is the one thing that edits — under the engine's strategy, unchanged: **one buffer, edits that only ever shrink it, one string at the end.** A string is decoded once; a `Uint8Array`/`Buffer` is edited **in place, in your own memory**, and `value()` hands back a zero-copy view of it. An edit that would need the buffer to grow throws `RangeError` *before* the first write. Every edit returns `this`.

| Method | Same as | Notes |
|---|---|---|
| `slice(start, end?)` | | keep `[start, end)` in units; out of range throws |
| `after` / `afterLast` / `before` / `beforeLast` `(target, length?)` | §1 | keep that piece; **a missing target is a no-op** — the chain's `''` |
| `remove(needles)` | §4 | |
| `squeeze(chars?, keep?)` | §6 | |
| `unescape(opts?)` | §10 | |
| `trim` / `trimStart` / `trimEnd` `(chars?)` | §10 | |
| `truncate` / `truncateBytes` / `truncateGraphemes` `(max, ellipsis?)` | §8 | an ellipsis must fit in the cut region |
| `dedent()` | §10 | |
| `wrap(width)` | §13 | the space before a word that no longer fits becomes `\n`; same length |
| `foldCase()` | | simple case folding; in utf-8 a fold that widens (`Ⱥ` → `ⱥ`) is refused up front |
| `toString()` / `value()` | | one string; or the source's own type — a zero-copy `subarray` for a buffer |
| `length` · `charCount` · `byteLength` · `graphemeCount` · `displayWidth` | §3 | getters; `length` is in units (code units or bytes) |
| `raw` | | the engine's `Mut` — a source for any `str-im` method |

```js
import { mutable } from '@slnknrr/str-ez';

mutable('  Héllo,,  wörld!!  ').trim().squeeze(',! ').remove('l').foldCase().toString(); // 'héo, wörd!'

const buf = Buffer.from('  héllo  ');
const m = mutable(buf).trim();       // edits buf itself
m.value();                           // Uint8Array(6) — a view into buf, no copy
m.value().buffer === buf.buffer;     // true
m.byteLength;                        // 6

mutable('世').truncateBytes(2, 'ab'); // RangeError before any write: 'ab' does not fit in 1 unit
mutable('abc').after('z').toString(); // 'abc' — not found, nothing happened, the chain went on
```

What is deliberately **not** on it: `escape`, `skeleton`, `deburr` and anything else that can grow. The strategy is the guarantee.

---

## TypeScript

Declarations ship with the package; nothing to install from `@types`. Option and result types are the engine's own, re-exported: `Source`, `Target`, `Mut`, `SplitOptions`, `PairOptions`, `NearOptions`, `EscOptions`, `GlobOptions`, `Indent`, `NamingStyle`, plus `Needles`, `IndexOptions`, `Run`, `MutableString<T>` (`T` follows the input, so `value()` is a `string` or a `Uint8Array` accordingly) and `StrEz` (the shape of the default export).

```ts
import ez, { truncateBytes, indexOfAll } from '@slnknrr/str-ez';
import type { Run, NamingStyle } from '@slnknrr/str-ez';

truncateBytes(buffer, 120);                  // Uint8Array in, Uint8Array out
indexOfAll('aaaa', 'aa', { overlapping: true });
ez.after = () => '';                         // error: the default export is frozen
```

`types/str-ez.d.ts` is written by hand and checked against `types/api.test-d.ts` with `npm run types`.

---

## When to reach for `str-im` instead

- You want to pull results one at a time and stop — an iterator, not an array.
- You need `-1` ("matched in full") as a distinct answer.
- You need a configured engine: a custom source type, a different vowel alphabet, extra confusables, a traced or specialized method via `_strim(overrides)`.

Both packages can be used side by side; `str-ez` never shadows anything in the engine.

---

## Scripts

| Command | Does |
|---|---|
| `npm test` | behavioral suite (`node --test`, zero dev dependencies) |
| `npm run types` | type-check the shipped declarations (`tsc --noEmit`) |

---

## Links

- Source: [https://codeberg.org/slnknrr/strez_lib.js-for-pm_npm/src/branch/main](https://codeberg.org/slnknrr/strez_lib.js-for-pm_npm/src/branch/main)
- npm: [https://www.npmjs.com/package/@slnknrr/str-ez](https://www.npmjs.com/package/@slnknrr/str-ez)
- The engine: [@slnknrr/str-im](https://www.npmjs.com/package/@slnknrr/str-im)

## Author

Yury Slinkin (Юрий Слинкин)

- Email: slnknrr.dev@gmail.com
- Codeberg: https://codeberg.org/slnknrr

## License

MIT. See [LICENSE.md](https://codeberg.org/slnknrr/strez_lib.js-for-pm_npm/src/branch/main/LICENSE.md).
