# Changelog

All notable changes to libharfbuzz-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.1 — 2026-09-18

The documentation and comments in plain prose; no declaration changed.

### Corrected against the HarfBuzz reference

- Three entry points carry `[io, ffi]`: one opens a file and two read
  the process locale.
- A vertical advance is negative for every font, because HarfBuzz
  counts the downward direction as negative.
- The cluster levels are 0 for monotone values grouped by grapheme, 1
  for monotone values that are not grouped, and 2 for values that are
  neither.
- `hb_font_create` takes a reference to the face, so releasing the
  face after creating the font is correct.

## 0.1.0 — 2026-09-16

The first release: sixty-seven entry points of the HarfBuzz C API, one
`@ffi` declaration each, and no logic.

### Added

- `libharfbuzz` — the whole surface, in seven groups.
  - The version and the enumerations as strings: `hb_version`,
    `hb_version_string`, `hb_tag_from_string`, `hb_tag_to_string`,
    `hb_direction_from_string`, `hb_direction_to_string`,
    `hb_script_from_string`, `hb_script_to_iso15924_tag`,
    `hb_language_from_string`, `hb_language_to_string` and
    `hb_language_get_default`.
  - The blob: `hb_blob_create`, `hb_blob_create_from_file_or_fail`,
    `hb_blob_get_length`, `hb_blob_get_data` and `hb_blob_destroy`.
  - The face: `hb_face_count`, `hb_face_create`, `hb_face_destroy`,
    `hb_face_get_glyph_count`, `hb_face_get_upem`,
    `hb_face_get_index`, `hb_face_reference_table` and
    `hb_face_get_table_tags`.
  - The font: `hb_font_create`, `hb_font_create_sub_font`,
    `hb_font_destroy`, `hb_font_get_face`, `hb_font_set_scale`,
    `hb_font_get_scale`, `hb_ot_font_set_funcs`,
    `hb_font_get_nominal_glyph`, `hb_font_get_glyph_h_advance`,
    `hb_font_get_glyph_v_advance`, `hb_font_get_glyph_extents`,
    `hb_font_get_h_extents`, `hb_font_get_glyph_name` and
    `hb_font_get_glyph_from_name`.
  - The buffer: `hb_buffer_create`, `hb_buffer_destroy`,
    `hb_buffer_reset`, `hb_buffer_clear_contents`,
    `hb_buffer_add_utf8`, `hb_buffer_add_codepoints`,
    `hb_buffer_get_length`, the setters and getters for the direction,
    the script and the language,
    `hb_buffer_guess_segment_properties`, the two cluster level calls,
    `hb_buffer_reverse`, `hb_buffer_has_positions`,
    `hb_buffer_get_glyph_infos` and `hb_buffer_get_glyph_positions`.
  - Shaping and serialisation: `hb_shape`, `hb_shape_full`,
    `hb_shape_list_shapers`, `hb_feature_from_string`,
    `hb_feature_to_string`,
    `hb_buffer_serialize_format_from_string` and
    `hb_buffer_serialize_glyphs`.
  - OpenType layout: `hb_ot_layout_has_substitution` and
    `hb_ot_layout_has_positioning`.
- `tests/libharfbuzz_tests.nv` — thirteen tests over the signatures.
  They call the C library, so they need HarfBuzz installed. The suite
  carries its own 776-byte TrueType font as hex and wraps it in a blob
  from memory, so it reads and writes nothing.

### The answer is two arrays

HarfBuzz gives a shaped run back as two arrays behind pointers, one of
glyph information and one of glyph positions, with one twenty-byte
record per glyph in each. Nothing is passed or returned by value, so
every shaping call binds. What the caller does by hand is walk the two
arrays at a stride of twenty bytes and read four-byte fields out of
them. The README's rules 6 and 7 carry the layouts.

### Named as missing

**Every entry point whose parameter or return type is a C `float`.** An
`@ffi` declaration has one floating point type, `Float`, and it lowers
to a C `double`. A C function reading a `float` parameter reads a
different 32 bits of the register, and gets 0.0 for every value whose
double has an empty low half — which is every round number. That
removes `hb_font_set_ptem` and `hb_font_get_ptem`,
`hb_font_set_synthetic_bold` and `hb_font_get_synthetic_slant`, and the
whole OpenType variable-font C API, because an `hb_variation_t`
carries a `float` value and `hb_font_set_var_coords_design` takes an
array of them. The defect is filed as
`an-ffi-float-is-lowered-as-a-c-double-so-a-c-function-taking-or-returning-float-gets-the-wrong-number`.

**The font functions.** `hb_font_funcs_create` and the twenty
`hb_font_funcs_set_*_func` calls install C function pointers that
answer a glyph's advance, extents and name. They are how a program
plugs its own font back end into HarfBuzz, and a novo-lang program
cannot produce one. `hb_ot_font_set_funcs` installs HarfBuzz's own
OpenType implementation and is here.

**The Unicode functions.** `hb_unicode_funcs_create` and its setters
are the same arrangement for the character properties. HarfBuzz's
built-in implementation is what a font gets by default.

**The buffer message callback.** `hb_buffer_set_message_func` reports
each step a shaper takes, and it is a function pointer.

**The draw and paint interfaces.** `hb_font_draw_glyph` and
`hb_font_paint_glyph` take a `hb_draw_funcs_t` and a
`hb_paint_funcs_t`, both structures of function pointers, and they are
how a caller gets a glyph's outline out of HarfBuzz.

**`hb_blob_create`'s destroy argument is declared and cannot be used.**
It is an `Int` in this package's signature, and the only value a
novo-lang program can pass for it is 0. Memory mode 0 copies the bytes,
so a program that passes 0 for both it and `user_data` owns its own
buffer and may free it.

**The subsetting interface.** The `hb_subset_*` calls are in the
separate library `libharfbuzz-subset`, and a `sys` package wraps one
library.

**`hb_buffer_diff`, `hb_buffer_deserialize_glyphs` and the segment
property record.** They are left out of the first release.

