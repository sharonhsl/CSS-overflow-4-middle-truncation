# [css-overflow-4] Middle Truncation

Authors:

- Sebastian Zartner (author of the [original proposal](https://github.com/w3c/csswg-drafts/issues/3937))
- Sharon Lam

## Introduction

Ellipsis in the middle of a text is a common truncation pattern, most useful when the beginning and end of a string matters. It is used commonly in technical identifiers to distinguish similar items, such as file names, URLs, file paths, and API keys.

**Example:** A file manager truncating long filenames:

| Filename                   | End truncation    | Middle truncation    |
| -------------------------- | ----------------- | -------------------- |
| `thesis-report-final.xlsx` | `thesis-report-…` | `thesis-…final.xlsx` |
| `thesis-report-draft.xlsx` | `thesis-report-…` | `thesis-…draft.xlsx` |

End truncation makes the two files appear identical; middle truncation preserves both ends, keeping the distinguishing suffix visible.

This document illustrates an update to the existing `text-overflow` spec to accommodate overflow handling via middle truncation.

## Goals

- Provide an ergonomic, backward-compatible interface to apply middle truncation to overflowing text.
- Allow authors to change the position of the default middle truncation overflow marker.

## Non-goals

- Describe how `text-overflow` middle truncation works with `line-clamp`.
- Describe how middle truncation should be implemented by browsers or platform.

## Current solutions

Many have attempted to address this either by applying `text-overflow` on split text across two DOM elements, or by using `ResizeObserver()` to detect overflow and format strings with JavaScript. However, these workarounds pose usability, performance, and accessibility concerns. Such problems are especially amplified in data-heavy displays like DataTable.

(TODO) More current JS solutions examples and their limitations.

## Proposed Solution

Extend the existing `text-overflow` shorthand to handle a `middle` position marker, with adjustable overflow marker position. Note that middle truncation should be an exclusive or with start and end ellipsis.

### Syntax

```ebnf
text-overflow = [ [ clip | <overflow-marker> ] && [ start | end ]? ]{1,2}
              | [ <overflow-marker> && [ middle | <length-percentage> ] ];
<overflow-marker>= [ ellipsis | <string> | fade | <fade()> ];
```

```css
/* middle ellipsis */
text-overflow: ellipsis middle;

/* middle ellipsis with offset */
text-overflow: ellipsis 30%;
text-overflow: ellipsis 3ch;
text-overflow: ellipsis calc(100%-4ch);

/* start and end with marker */
text-overflow: clip start ellipsis end;
text-overflow: clip ellipsis;

/* only end */
text-overflow: ellipsis;
text-overflow: ellipsis end;

/* only start */
text-overflow: ellipsis start;

/* invalid entry */
text-overflow: clip start ellipsis start;
text-overflow: fade end clip end;
text-overflow: middle;

/* Existing start and end syntax remains accepted */
text-overflow: clip;
text-overflow: ellipsis ellipsis;
text-overflow: ellipsis " [..]";
```

### Positioning

`<length-percentage>` specifies the overflow marker position measured against the available inline space within a line box, with the offset starting from the line-start edge. Such inline space excludes any portion that overflows out of the block container. Values less than `0` are clamped to `0`. Values greater than the inline space are clamped to the extent of the space. Percentages resolve against the available inline space. The keyword `middle` is equivalent to `50%`.

Below are some basic examples of offset calculation:

```
// ruler for a 40ch wide mono-space container
0123456789012345678901234567890123456789

├──── 20 chars ────┤├──── 20 chars ────┤
The quick brown fox …g by the river bank

├─12 chars ┤├──────── 28 chars ────────┤
The quick br… lazy dog by the river bank

├3┤├─────────── 37 chars ──────────────┤
The …over the lazy dog by the river bank

├─────────── 36 chars ─────────────┤├─4┤
The quick brown fox jumps over the l…ank

```

```html
<div class="container">
  <p class="truncate middle">
    The quick brown fox jumps over the lazy dog by the river bank
  </p>
  <p class="truncate offset-30">
    The quick brown fox jumps over the lazy dog by the river bank
  </p>
  <p class="truncate offset-3ch">
    The quick brown fox jumps over the lazy dog by the river bank
  </p>
  <p class="truncate offset-end">
    The quick brown fox jumps over the lazy dog by the river bank
  </p>
</div>
```

```css
.container {
  width: 40ch;
  font-family: ui-monospace, monospace;
  font-size: 16px;
}

.truncate {
  white-space: nowrap;
  overflow: hidden;
}

.middle {
  text-overflow: ellipsis middle;
}

.offset-30 {
  text-overflow: ellipsis 30%;
}

.offset-3ch {
  text-overflow: ellipsis 3ch;
}

.offset-end {
  text-overflow: ellipsis calc(100% - 4ch);
}
```

#### With bidirectional text

```
// brainstorming some bidi text

"تقرير_السنوي.docx"
"report_تقرير_final.pdf"
"تقرير_2024_annual_report.pdf"
"report_تقرير.pdf" vs. "تقرير_report.pdf"
"file_name.תקציר"
"draft(تقرير).docx"
```

#### With Inline-block

#### With small line box

#### With vertical direction text

#### With invalid entry

Though the syntax permits, repeated declaration of `start` or `end` marker will be regarded the same as not specifying `text-overflow`.

```css
text-overflow: clip start ellipsis start;
text-overflow: fade end clip end;
```

## Pain points

###

## Future work

### With `line-clamp`

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Florian Rivoal
- Andreu Botella
- [etc.]
