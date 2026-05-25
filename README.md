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

- Describe how middle truncation works with multi-line truncation, i.e. `line-clamp`.
- Describe how middle truncation should be implemented by browsers or platform.

## Current solutions

Many have attempted to address this either by applying `text-overflow` on split text across two DOM elements, or by using `ResizeObserver()` to detect overflow and format strings with JavaScript. However, these workarounds pose usability, performance, and accessibility concerns. Such problems are especially amplified in data-heavy displays like DataTable.

(TODO) More current JS solutions examples and their limitations.

Browser engines currently already use middle truncation for selected file names in the `<input type="file">` element.

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

`<length-percentage>` specifies the overflow marker position measured against the available inline space within a line box, with the offset starting from the line-start edge. Such inline space excludes any portion that overflows out of the block container. Values less than `0` are clamped to `0`. Values greater than the inline space are clamped to the extent of the space. Percentages resolve against the available inline space.

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

The keyword `middle` is generally equivalent to `50%`.
Though user agents are allowed to position the overflow marker intelligently, so that important parts of the text stay visible as far as possible. Examples for that might be the file extension in a file path or the domain and the final non-path part of a URL.

#### With invalid entry

Though the syntax permits, repeated declaration of `start` or `end` marker will be regarded the same as not specifying `text-overflow`.

```css
text-overflow: clip start ellipsis start;
text-overflow: fade end clip end;
```

## Layout behavior

### Scrolling

Like end- and start-truncation, middle truncation is a purely visual effect that has no influence on the layout. This is done by making the end portion of the text visually fixed, while the UA may make the start portion scrollable to reveal different parts of the hidden content, or alternatively, the entire line may remain non-scrollable.

#### Example: Scrollable middle truncation

Consider a file path that is truncated in the middle with a scrollable start portion:

```html
<div class="file-path-container">
  <p class="truncate-middle">
    /home/user/documents/projects/important-project-name/subfolder/very-long-file-name.txt
  </p>
</div>
```

```css
.file-path-container {
  width: 50ch;
  font-family: monospace;
  overflow: hidden;
}

.truncate-middle {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis middle;
}
```

**Visual result:**
```
/home/user/documents/…very-long-file-name.txt
```

If the user agent supports scrolling to reveal truncated content, the start portion of the path could be scrolled horizontally to reveal:

```
/projects/important-project-name/subfolder…very-long-file-name.txt
```

while keeping the end portion (filename) fixed for reference.

### Intrinsic Sizes
Middle truncation does not affect intrinsic sizes. Meaning, when `text-overflow: ellipsis middle` is applied, properties such as  `min-content` and `max-content` are unaffected. In other words, truncation should happen only after inline sizes have been resolved. This mirrors how the current start/end truncation work, avoiding circular dependencies between layout box and line box calculation.

This main benefit is that middle truncation becomes safe to apply in any layout context without affecting the layout calculation, i.e. a flex item, a grid cell, a table cell, can still compute the same way. Putting this into context, adding middle truncation to an existing layout cannot cause sibling elements to resize, line counts elsewhere on the page to change, or scrollbars to appear on ancestor containers.


#### Example: middle truncation does not affect intrinsic sizing

```html
<div class="container">
  <p class="filename">
    Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
  </p>
  <p class="label">Short text</p>
</div>
```

```css
.container {
  display: flex;
  gap: 1rem;
  width: 50ch;
  font-family: monospace;
}

.filename {
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis middle;
}
```

The flex algorithm sizes the children using their full content widths. "Short text" takes its content size (10ch); the long text takes the remaining inline space via `flex: 1`. Truncation then activates within the width the long text was assigned. Removing or changing the `text-overflow` value does not resize "Short text", because the flex layout was determined by intrinsic sizes independent of truncation. 

**Visual result:**
```
Lorem ipsum dolor…re magna aliqua. Short text
```

### With Inline-block

### With small line box

When the available inline space within linebox becomes to small for the prefix, overflow marker, and suffix, the priority of elements to display should be the following:

1. Overflow marker
2. Prefix
3. Suffix

Below is an example showing a truncated with middle ellipsis as its container shrinks:

```
Lorem Ipsum is simply dummy text of the printing and typesetting industry.
Lorem Ipsum …industry.
Lorem…
…
```

### With vertical direction text

## Interaction with bidirectional text

To maintain intuitiveness of offset calculation, we should use line-start position as the reference point for bidi text as well. Below are examples demonstrating why this design decision is ergonomic.

Example 1 and 2 shows the base cases of LTR and RTL based paragraphs with occasional strings of the opposite direction. Example 3 is a list of bidi filenames in LTR direction, to showcase the flexibility of using line-start as reference point for offset.

### Example 1: LTR paragraph with RTL content

```html
<p>
  The title is
  <cite dir="rtl">AN INTRODUCTION TO <span dir="ltr">c++</span></cite>
  in arabic.
</p>
```

The original sentence and the truncated text are:

> <div dir="ltr">The title is مدخل إلى C++ in Arabic.</div>
> <div dir="ltr">The title …Arabic.<div>

### Example 2: RTL paragraph with LTR content

```html
<p dir="rtl">W3C מעביר את שירותי הארחה באירופה ל - ERCIM.</p>
```

The original sentence and the truncated text are:

> <div dir="rtl">W3C מעביר את שירותי הארחה באירופה ל - ERCIM.</div>
> <div dir="rtl">W3C מעביר… באירופה ל - ERCIM.</div>

### Example 3: Technical identifiers like filenames

Filenames frequently mix scripts but are typically displayed in `ltr` direction for consistency with file system conventions, even when they contain RTL characters. Users can override the direction with the `dir` attribute on the block container when needed.

| Filename                                            | Middle Truncated                    | Notes                                                |
| --------------------------------------------------- | ----------------------------------- | ---------------------------------------------------- |
| `report_تقرير_final.pdf`                            | `report_…l.pdf`                     | LTR-wrapped RTL in the middle                        |
| <div dir="rtl">`تقرير_السنوي.docx`</div>            | <div dir="rtl">`تقرير_….docx`</div> | <div dir="ltr">All-RTL name with LTR extension</div> |
| <div dir="ltr">`تقرير_السنوي.docx`</div>            | <div dir="ltr">`تقرير_….docx`</div> | All-RTL name with LTR extension in a LTR container   |
| <div dir="ltr">`تقرير_2024_annual_report.pdf`</div> | `2024….t.pdf`                       | RTL, Western digits, and LTR                         |
| `report_تقرير.pdf`                                  | `report….pdf`                       | LTR start, RTL middle, LTR extension                 |
| `file_name.תקציר`                                   | `file_…תקציר`                       | Latin name with RTL extension (extension is RTL)     |
| `draft(تقرير).docx`                                 | `draft(….docx`                      | Neutral parentheses around an RTL                    |

Each of these strings exercises a different combination of LTR, RTL, and neutral (punctuation, digit) characters.


## Further notes

`text-overflow` is kept applying to block containers only for the time being.

## Open questions

* How does it interact with everything else? Intrinsic sizes, floats, margins, bidi, display…
* Enforce creation of a BFC?

## Future work

### With `line-clamp`

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Florian Rivoal
- Andreu Botella
- Emilio Cobos Álvarez
- [etc.]
