# libharfbuzz-sys

HarfBuzz is a text shaping engine. It takes a run of Unicode text and a
font, and answers the sequence of glyphs that draws that text and where
each one goes. It is documented in the
[HarfBuzz manual](https://harfbuzz.github.io/), and it is the shaper
behind Firefox, Chrome, Android, LibreOffice and most of the Linux
desktop. This package declares sixty-seven of that library's entry
points to novo-lang, one declaration each.

**Status: a binding, not a port.** Every function in this package is a
declaration of a function in HarfBuzz. The package contains no logic of
its own, and it does nothing without the C library installed. The
sixty-seven entry points are the ones a program needs to load a font,
shape a run and read the result; the section "What is not included"
says what a program still cannot do with them alone.

## What it is

**Shaping** is the step between text and drawing. One character may be
drawn by several glyphs, several characters by one glyph, and the
choice depends on the characters around them, on the script, on the
language and on what the font offers. Shaping is what decides which
glyphs and in what order.

A **glyph** is one drawn shape in a font, named by an index rather than
by a character. A **cluster** is the group of characters one or more
glyphs came from. HarfBuzz reports a cluster number per glyph, and a
text editor uses it to put the caret in the right place.

A **blob** is a block of bytes with a reference count. A **face** is
one font inside a blob. A **font** is a face at a scale. A **buffer**
holds the text going in and the glyphs coming out, and the same buffer
is reused run after run.

The **segment properties** are the direction, the script and the
language of a run. Shaping needs all three. A program that knows them
sets them; a program that does not calls
`hb_buffer_guess_segment_properties`, which reads the script off the
characters, the direction off the script and the language off the
process locale.

A **feature** is an OpenType switch: `kern` for kerning, `liga` for
ligatures, `smcp` for small capitals. A font declares which it has, the
shaper turns a default set on, and a caller may turn any of them on or
off over the whole run or over a range of it.

A **tag** is four characters read as one 32-bit number, big-endian.
OpenType names a table, a feature, a script and a language system that
way, and so does HarfBuzz.

## Install

```
novo pkg add libharfbuzz-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its headers come from the system package
`libharfbuzz-dev`:

```
sudo apt install libharfbuzz-dev
```

On macOS the Homebrew formula is `harfbuzz`. On other systems the
library builds from the HarfBuzz source with Meson.

## Example

Two characters shaped into two glyphs:

```novo ignore
use libharfbuzz

fn main() [io, fs, ffi]
    // The font file, wrapped in a blob that copies it.
    let font_bytes = ptr.alloc(65536)
    let n = 65536
    let blob = libharfbuzz.hb_blob_create(font_bytes, n, 0, 0, 0)
    let face = libharfbuzz.hb_face_create(blob, 0)
    let font = libharfbuzz.hb_font_create(face)

    // Sixteen pixels in 26.6 units, so every advance comes back in 26.6.
    libharfbuzz.hb_font_set_scale(font, 16 * 64, 16 * 64)

    // The text, and the three properties shaping needs.
    let buffer = libharfbuzz.hb_buffer_create()
    libharfbuzz.hb_buffer_add_utf8(buffer, "Hello", -1, 0, -1)
    libharfbuzz.hb_buffer_guess_segment_properties(buffer)
    libharfbuzz.hb_shape(font, buffer, 0, 0)

    // Two arrays of twenty-byte records, one record per glyph.
    let length = ptr.alloc_word()
    let infos = libharfbuzz.hb_buffer_get_glyph_infos(buffer, length)
    let positions = libharfbuzz.hb_buffer_get_glyph_positions(buffer, length)
    let count = ptr.read_word(length) & 4294967295

    for i in 0..count
        let glyph = ptr.read_word(infos + i * 20) & 4294967295
        let cluster = ptr.read_word(infos + i * 20 + 8) & 4294967295
        let advance = ptr.read_word(positions + i * 20) as i32
        println("glyph ${glyph} from character ${cluster}, ${advance / 64} pixels wide")

    libharfbuzz.hb_buffer_destroy(buffer)
    libharfbuzz.hb_font_destroy(font)
    libharfbuzz.hb_face_destroy(face)
    libharfbuzz.hb_blob_destroy(blob)
    ptr.free(length)
    ptr.free(font_bytes)
```

The example is fenced as an illustration rather than a compiled block
because `novo doc` compiles the blocks in documentation comments and not
the ones in this file. The same calls are in
`tests/libharfbuzz_tests.nv`, where the numbers are asserted.

## What the package contains

| Module | Contents |
| --- | --- |
| `libharfbuzz` | Every entry point, in seven groups: the version and the enumerations as strings, the blob, the face, the font, the buffer, shaping and serialisation, and the two OpenType layout questions. |

The seven groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Version and enumerations | 11 | Reports the version, and turns tags, directions, scripts and languages between their text and their numbers. |
| The blob | 5 | Wraps bytes, or a file, in a reference-counted block. |
| The face | 8 | Opens one font in a blob and reports its glyph count, its scale and its tables. |
| The font | 14 | Scales a face and answers a glyph for a character, its advances, its extents and its name. |
| The buffer | 20 | Holds the text and its properties, and gives the shaped glyphs back as two arrays. |
| Shaping | 7 | Shapes a buffer, chooses the features, and writes the result out as text. |
| OpenType layout | 2 | Asks whether a face has a substitution or a positioning table. |

## How to choose an entry point

`hb_blob_create` is for bytes already in memory and
`hb_blob_create_from_file_or_fail` is for a file. The file form is the
only call in the package with an `[io]` effect besides the two that
read the process locale.

`hb_buffer_add_utf8` is for text still encoded and
`hb_buffer_add_codepoints` for text already decoded. Both take an item
offset and length that select the part to shape while the rest stays
readable, which is what lets a shaper join a letter to a neighbour that
is not in the run.

`hb_shape` is the ordinary call. `hb_shape_full` is the same call with
a list of shapers to try and an answer saying whether one worked; a
program uses it to insist on `"ot"` and refuse the fallback shaper.

`hb_buffer_clear_contents` is for the next run: it drops the text and
the properties that describe it, and keeps the cluster level and the
flags. `hb_buffer_reset` puts the buffer back to the state it was
created in.

## The rules a user needs

1. **A pointer is an `Int`, and zero is null.** Every handle the C
   library returns arrives as the address it returned.
2. **Everything but a language is reference counted, and everything
   but a language is released.** A blob by `hb_blob_destroy`, a face by
   `hb_face_destroy`, a font by `hb_font_destroy` and a buffer by
   `hb_buffer_destroy`. A face holds a reference to its blob and a font
   to its face, so releasing the blob right after creating the face is
   correct. A language handle is interned for the life of the process
   and is never released.
3. **A failure is an empty object, not a null.** `hb_face_create` over
   bytes that are not a font answers a face with no glyphs, and
   `hb_face_reference_table` for a table the font does not have answers
   a blob of length 0. `hb_blob_create_from_file_or_fail` is the one
   call here that answers 0, which is what the `_or_fail` in its name
   means.
4. **An answer that is a position is a signed 32-bit number, and some
   of them are negative.** Bind the answer to a local with `as i32`
   before comparing it with zero. `hb_font_get_glyph_v_advance` is
   negative for a font with no vertical metrics, a glyph's extents
   height is negative because the box is measured downwards, and a
   font's descender is negative.
5. **A scale is a multiplier, not a size.** Every position a font
   answers is the design-unit value times the scale divided by the
   face's units per em. A scale equal to the units per em answers
   design units; a scale of the pixel size times 64 answers 26.6
   pixels. A font starts at the face's units per em.
6. **A glyph information record is twenty bytes.**

   | Offset | Width | Field |
   | --- | --- | --- |
   | 0 | 4 | the glyph index |
   | 4 | 4 | a mask, private to the library |
   | 8 | 4 | the cluster |
   | 12 | 8 | private |

7. **A glyph position record is twenty bytes**, and every field in it
   is signed.

   | Offset | Width | Field |
   | --- | --- | --- |
   | 0 | 4 | the x advance |
   | 4 | 4 | the y advance |
   | 8 | 4 | the x offset |
   | 12 | 4 | the y offset |
   | 16 | 4 | private |

   The offset moves the glyph without moving the pen, which is how a
   mark is attached. The advance moves the pen.

8. **The two arrays are as long as each other and belong to the
   buffer.** They stop being valid when the buffer is shaped again,
   cleared, reset or released.
9. **The length of a buffer is characters before shaping and glyphs
   after it.** They are not the same number: a ligature makes it
   smaller, a decomposition makes it larger.
10. **A buffer cannot be shaped without a direction.** Set one, or call
    `hb_buffer_guess_segment_properties`, which fills in whichever of
    the direction, the script and the language is still unset.
11. **The direction values are 4, 5, 6 and 7**, for left to right,
    right to left, top to bottom and bottom to top. 0 is the invalid
    direction, which is what an untouched buffer has.
12. **A script and a tag are the same kind of number.** Four characters
    read big-endian. `hb_script_from_string("Latn", -1)` and
    `hb_tag_from_string("Latn", -1)` answer the same value.
13. **`hb_tag_to_string` writes four bytes and no terminator.** Read
    them with `ptr.read_bytes_n(buf, 4)`. A tag shorter than four
    characters is padded with spaces at the other end.
14. **A feature record is sixteen bytes**: the tag at 0, the value at
    4, the start at 8 and the end at 12, each four bytes. A feature
    with no range written covers 0 to 4294967295, which is the whole
    run.
15. **`hb_buffer_serialize_format_from_string` does not check the
    name.** It upper-cases what it is given and reads it as a tag, so
    an unknown name answers a tag that is not a format. Only an empty
    name answers 0.
16. **Memory mode 0 copies the bytes.** `hb_blob_create` with mode 0
    lets the caller free its own buffer as soon as the call returns.
    Modes 1, 2 and 3 borrow them, and the caller must then keep them
    alive for the life of the blob.
17. **The `user_data` and `destroy` arguments of `hb_blob_create` are
    for a C caller.** A novo-lang program passes 0 for both, because
    `destroy` is a C function pointer.

## What is not included

- **Every entry point whose parameter or return type is a C `float`.**
  An `@ffi` declaration has one floating point type, and it is a C
  `double`. That leaves out `hb_font_set_ptem`, `hb_font_get_ptem`,
  `hb_font_set_synthetic_bold`, `hb_font_get_synthetic_slant`, and the
  whole OpenType variable-font interface, because a variation carries
  a `float` value.
- **The font functions.** `hb_font_funcs_create` and the twenty
  `hb_font_funcs_set_*_func` calls install C function pointers that
  answer a glyph's advance, extents and name, and the novo-lang foreign
  function interface passes integers, floats and strings.
  `hb_ot_font_set_funcs` installs HarfBuzz's own OpenType
  implementation and is here.
- **The Unicode functions.** `hb_unicode_funcs_create` and its setters
  are the same arrangement for the character properties. The built-in
  implementation is what a font gets by default.
- **The buffer message callback.** `hb_buffer_set_message_func` reports
  each step a shaper takes, and it is a function pointer.
- **The draw and paint interfaces.** `hb_font_draw_glyph` and
  `hb_font_paint_glyph` take structures of function pointers. They are
  how a caller would get a glyph's outline out of HarfBuzz;
  `libfreetype-sys` is the other way to the same outline.
- **The subsetting interface.** The `hb_subset_*` calls are in the
  separate library `libharfbuzz-subset`, and one `sys` package wraps
  one library.
- **`hb_buffer_diff` and `hb_buffer_deserialize_glyphs`.** They are
  left out of the first release.

## Related packages

There is no HarfBuzz port in novo-lang, and there is no plan for one.
Shaping is a body of script-specific knowledge rather than an algorithm
with a specification, and HarfBuzz is where that knowledge lives.

`libfreetype-sys` binds FreeType, which draws the glyphs this package
chooses. The two are used together: HarfBuzz answers glyph indices and
positions, and FreeType turns an index into a bitmap. Neither replaces
the other.

`font-nv` is the font port written in novo-lang. It reads outlines and
metrics out of a font file, which is a different job from shaping.
`font-nv` is planned and not published yet.

## Tests

`tests/libharfbuzz_tests.nv` holds thirteen tests written against the
signatures. They call the C library, so `novo test` needs HarfBuzz
installed and linkable:

```
novo test tests/libharfbuzz_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite carries its own font as hex: a 776-byte TrueType font with
three glyphs, a square named `square` mapped from U+0041 and a triangle
named `triangle` mapped from U+0042 beside the substitute glyph. It is
wrapped in a blob from memory, so the suite reads and writes nothing.

The tests assert the tag, direction, script and language round trips,
that a blob with memory mode 0 holds a copy rather than the caller's
bytes, the face's glyph count and its ten table tags, that a font
starts at the face's units per em and that a negative scale arrives
negative, the glyph a character maps to and the doubled advance at
twice the design scale, the negative vertical advance and the negative
extents height, what `hb_buffer_clear_contents` keeps that
`hb_buffer_reset` does not, the two glyphs and two clusters `"AB"`
shapes into with their positions read at a stride of twenty bytes, a
feature read from `"kern=0"` and written back as `"-kern"`, and the
serialised form `[square=0+1400|triangle=1+1400]`.

`novo --leak-check` reports twelve leaked objects at the end of the
run. They are the `ptr.read_str` and `ptr.read_bytes_n` copies the
tests make out of the library's own strings; both are declared
untracked, which is a defect in the toolchain and not in this package.

## Implementation status

| Group | State |
| --- | --- |
| Version and enumerations | Complete. |
| The blob | Complete for the create, the two accessors and the release. |
| The face | Complete for the read-only interface. |
| The font | Complete for the OpenType glyph functions. |
| The buffer | Complete for text in and glyphs out. |
| Shaping | Complete, with the features and the two serialisation calls. |
| OpenType layout | The two questions a caller asks before shaping. |
| The point size and the synthetic styles | Absent. Their arguments are C `float`. |
| Variable fonts | Absent. A variation carries a C `float`. |
| The font and Unicode functions | Absent. They are C function pointers. |
| The draw and paint interfaces | Absent. They are structures of C function pointers. |
| Subsetting | Absent. It is a second library. |
| The face builder | Absent. Left out of the first release. |

## Licence

Apache-2.0. See [LICENSE](LICENSE).

HarfBuzz itself is distributed under the MIT licence, and installing it
is the reader's own step.
