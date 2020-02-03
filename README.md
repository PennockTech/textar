textar: text archival format
============================

Textar is a human-editable single-file archive format for holding many other
files.  It can embed binary files, via Base64, but that is very much not the
optimized use-case.

To backup arbitrary content, use something else.

To create a backup which can be committed into revision control, consider
textar.

textar should play cleanly with compression and encryption, and be entirely
agnostic of such things at the file format layer.

Suggested MIME type is: `application/x-textar`

Suggested file extension is: `.textar`

Note: the LICENSE file here pertains to any software in this repository unless
indicated otherwise, and to the text of this pseudo-specification.  The
specification may be freely implemented by others.  Please at least try to
stay compatible and work with us if incompatible changes are needed.


### Contents

* Tools
* File Format
* Examples
* Inspiration


## Tools

Run:

    go build -v ./go/...

The available tools are: **TBD**

Contributions of ports of tools to other languages are typically welcome.


## File Format

A primary goal is convenience for humans, above convenience for smallest set
of necessary features.  Tools have some choices for how to encode entries.

`textar` files are line-based, with `<NL>` as a line terminator.  All lines
end with 0x0A (`\n`), including the last line.  Parsers should ignore trailing
whitespace in metadata and header lines, so that CRLF can be used.

The very first line is constrained to be `US-ASCII` and can specify a default
character set for all that follows, but UTF-8 is the assumed default.

Metadata lines are "streaming JSON" or "line JSON", as per
`application/x-jsonlines` or `application/ld+json`.

The first line is for archive control and provides for file-format
identification, with a fixed initial 20 octets.

There is a feature-controlled optional second archive control line.

Afterwards are pairs of line-JSON headers and content, repeated as many
times as needed.  The content has one extra blank line after it.

Optionally, there is a final index of LINE-JSON, to aid tools with seeking.

All top-level keys in all JSON headers MUST be US-ASCII.

All header lines should always be parsed to ignore any whitespace after the
closing brace `}` and before the newline `\n`.

JSON control lines should be no longer than 4096 octets.  An extension which
requires more data than this should use a feature to enable extra control
lines for an entry, in a similar style to `Line2control` below.

There is no inherent length limit upon the non-JSON lines for content within an
archive.  Base64-encoded data MUST use lines no longer than 76 characters, and
tools are free to optimize for that and reject longer lines.  For non-binary
files represented normally, the format itself does not impose a limit.
However, tools are encouraged to impose limits to avoid denial-of-service
attacks.

If, at textar creation time, a tool detects lines longer than 1000 characters,
then the `longlines` control SHOULD be set on that line to provide a hint
to readers.  Implementations MAY reject lines longer than 4096 octets
(including final newline) if the `longlines` hint is not used, and point
complainants at this specification to push ire towards the creator tools.
A `longlines` hint may still be subject to "don't be stupid" limits.  Given
the optimized use-cases of the textar format, geared towards archiving small
files, an upper limit of 1MiB in size for long lines seems entirely
reasonable.


### First Line: Archive Control

The first line MUST START: `{"format":"textar/1"`
Those 20 octets should be there, with no added whitespace, to make it easier
for generic tools to identify the file-format.  Extra whitespace before the
closing brace `}` is strongly discouraged.

This first line is for Archive Control and is about the textar archive itself,
rather than entries.  This first line MUST be entirely in US-ASCII.

Options in this Archive Control line can change the character set used for all
lines thereafter; notably, this means that _values_ in the header lines might
not be UTF-8 despite being in a JSON structure.  This is not normal JSON and
the first-pass tooling is unlikely to support this.  Note that the _whole
point_ of this file-format is to have something which humans can edit, so all
lines must be in one consistent file-format.  The JSON-ish lines can not be
excluded from this requirement just because standalone JSON has to be UTF-8.

A minimal first line is:

    {"format":"textar/1"}

This is equivalent to:

    {"format:"textar/1","encoding":"UTF-8","newlines":"\n"}

Allowed fields:

* `format`: mandatory string, must be first; fixed value for this version
  of the specification.
* `encoding`: optional string, defaults to `"UTF-8"`.
* `newlines`: optional string, defaults to `"\n"`; any common platform line
  endings are permitted, provided that the last character is `\n`.  Changing
  this value controls lines in content sections.  The header sections are
  agnostic as to extra whitespace.
* `creation_date`: optional string, an RFC 3339 profile of ISO 8601
  timestamps.
* `features`: optional array of non-empty strings, enabling extension
  features.

Parsers should ignore any extra fields and only warn on their presence if
asked to be particularly insightful as to structure, so that
backwards-compatible extensions can be added.  Such extensions SHOULD be
indicated with a string in the `features` array.

There is no central registry for feature extensions.  Implementers should
exercise their own discretion in best avoiding namespace collisions.  Note
that the first line, and thus all feature strings, is constrained to be
US-ASCII.

If the first character of a feature is a lower-case ASCII letter, then the
feature can safely be ignored by other tools without affecting basic parsing.
If the first character of a feature is an upper-case ASCII letter, then the
feature affects basic parsing and tools should proceed with caution (dropping
to a read-only mode would be a good idea).
The base specification includes one example of each.
The feature name does not need to start with a letter.  No special meaning is
ascribed at this time to features starting with anything other than a letter.

Leading and trailing whitespace in feature names is discouraged.
Representations of non-printing characters is discouraged.

An example first-line with non-standard settings, here shown with inserted
whitespace to lay things out more clearly for humans, might be:

    {
      "format":"textar/1",
      "encoding": "ISO-2022-JP",
      "newlines": "\r\n",
      "creation_date": "2020-02-02T14:00:00Z",
      "features": ["Line2control", "vnd/yoyodyne/foo:42"]
    }

#### Known Features

1. **`Line2control`**
2. **`index`** and **`index-stale`**

Feature `Line2control`:  
There is a second archive-control line, before the content-pairs start.
This second line IS NOT CONSTRAINED TO US-ASCII.
This second line, like all lines after the first, is in the encoding
controlled by the first line: a default of `UTF-8`.
There is no current use for this feature; it exists to provide an escape hatch
from the US-ASCII constraints of the first line.

Feature `index`:  
There is a final entry in the file which is an index.  The content of this
index is TBD FIXME FIXME.  If editing the file such as to change offsets,
polite tools will don't handle the index themselves will switch the feature to
`index-stale` to indicate that any offsets in the offset can no longer be
trusted.


### File Entries

Each file entry begins with one line of line-JSON.  The character set of this
line is as controlled in the first Archive Control line.  Following the
line-JSON is the content, as controlled by the below, then one blank line.

Only the `filename` field is mandatory.

Fields:
* `filename`: string, mandatory, non-empty, may not contain a representation
  of ASCII NUL in the encoding or any escaping used.  Must be a complete
  string.  FIXME: TBD: NORMALIZATION
* `prefix`: optional non-empty string, default `"X"`; a prefix starting with
  an opening brace is forbidden (no "{" or "{anything").
* `base64`: boolean (`true` or `false` without quotes); default false.
* `longlines`: optional integer, longest length of a line seen in the file
* `jsonline`: optional boolean, default false
* `jsonmulti`: optional boolean, default false
* `type`: optional string
* `aclunix`: optional string, a traditional Unix basic file access ACL;
  default "create non-executable subject only to umask".
* `owner`: optional array of items
* `aclposix1e`: array of strings, representing entries in a POSIX.1e ACL
* `aclnfsv4`: array of strings, representing entries in an NFSv4 ACL
* `xattr`: JSON objects representing small extended attributes.  The objects
  SHOULD have string values for each key and tooling MAY protest objects which
  do otherwise.
* `xattr:large`: TBD FIXME FIXME
* `features`: array of feature strings, for extension

Following the line-JSON is the data:

 * by default, all lines are prefixed with a sequence of characters, "X"
   unless specified
 * if the `base64` boolean is true, lines are in base64 format with no prefix
   (and no PEM markers or other decoration)
 * if the `jsonline` boolean is true, the content is one line, itself starting
   with a `{` because it's a second JSON line.
 * if the `jsonmulti` boolean is true, the first line of the content shall be
   `"{\n"` (without quotes) (with variants for other whitespace before EOL per
   Archive Control line), and the last line of the content shall be `"}\n"`
   (without quotes); all intervening lines MUST be indented with whitespace.

At most one of the [`base64`, `jsonline`, `jsonmulti`] options may be true.

Lines of regular text are expected to be "short", constrained by the default
of 4096 octets ("bytes" these days) unless the `longlines` hint is used to
allow extractors to set up some other optimization.  Note that `longlines`
SHOULD be set if any line is longer than 1000 characters.  Note that
characters are not octets.  999 characters encoding to 4095 bytes does not
strictly need the `longlines` hint present.

After the data is either end-of-file or one blank line before the next file
entry (or end-of-file).  Tools should allow for a blank line at the end of a
file having been automatically removed by other tools, but the blank line is
technically a part of the entry.

Between the base64-or-prefix encoding of entries, and the line-JSON to open,
then assuming sane prefix values each file entry in a textar archive will
constitute one paragraph in a text-editor.

Blank lines other than the end-of-file indicator are not allowed, unless crazy
prefix controls have introduced such.  They may creep in or be removed, or end
up with extra whitespace in them, depending upon what crazy humans do when
editing textar files manually, so tools should aim to be "a little resilient"
to variances here, instead of rejecting outright based on missing or extra
empty, or contains-only-whitespace, lines.

In base64-mode, an opening brace `{` is not a valid character, so the next
entry begins on a line starting with an opening brace and there can be no
confusion.  No PEM markers (`-----` lines) are used for base64 data.

In normal mode, the prefix comes before each and every line of content.  The
default is `X`.  Because prefices starting with an opening brace are
forbidden, distinguishing extraneous blank lines followed by more data, from
the next file entry, is provable one way or the other.  When `jsonline` or
`jsonmulti` has been set, the content is one JSON object, and multi-line
entries are indented, so this distinction remains provable.

A non-empty line not starting with an opening brace or the prefix character or
whitespace (for `jsonmulti`) is a syntax error.
Beware that an extension might change this.
FIXME: this was the point at which I remembered xattr and large xattr
handling.  Should figure out what to do here.  Header or Trailer, what format?
A second prefix character?  A pair?

FIXME Perhaps:

    +user.mime_type
    :text/html
    X<html><body>foo</body></html>

FIXME Perhaps ... Internet Message Format header system?  Extension?

If the `type` is present, then use of an unrecognized or impermissible value
SHOULD by default be treated as a verbose "skip", yielding warnings and
ignoring the content but otherwise not erroring.

Values of `type`:

* `"file"`: the default
* `"skip"`: ignore this entry
* `"directory"`: there should be no actual content for this type; there may be
  attributes of various kinds, including ACLs and xattrs, as well as
  permissions.
* `"symlink"`: SHOULD be paired with `"jsonline": true`; content is expected
  to have two keys: `linkname` and `target`.
* Values containing a SOLIDUS (`/`, 0x2F) are reserved to indicate a MIME type
  which affects how the file should be extracted.
* Other values should not typically be implemented, but might be of use for
  system bootstrap in trusted contexts; a `features` extension should be used
  to effectively provide a namespace and interpretation for these.

Note that MIME type representation of regular files is a matter for
`user.mime_type` xattr data, not for the metadata of textar itself.  The
`type` field is explicitly not for letting arbitrary MIME types be set on
regular files.  Instead, a MIME type indicates that an entry is not a file to
be extracted to the file-system but is instead for dispatch according to some
logic expected to exist in the receiving system.  Conceptually, think of a
cloud-init MIME type, and which tools receive some directives.

A top-level MIME type of `signature` is hereby defined to exist in the context
of textar files and indicates extra verification may be performed.


#### Validation of metadata

Filenames in UTF-8 might be incomplete strings, truncated, or barred by
various normalization profiles.  Parsers should handle this and reject entries
which do not conform.  "Fixing up" should not be the default, that way lie
security problems when part of a larger system.

Filenames SHOULD use the SOLIDUS (`/`, 0x2F) as a directory separator, unless
changed with a feature at some level.  Filenames MUST NOT contain ASCII NUL.

Filenames which represent paths outside the current working directory SHOULD
be rejected by default.

Filenames SHOULD be unique within one textar archive.  Absent explicit
indication to the contrary, extractors SHOULD abort on seeing a repeated
filename.  An append-style archive to overwrite earlier entries MAY be used
but this SHOULD be explicitly requested and MUST NOT be demanded of
extractors.

Extractors SHOULD push all symbolic links into a pool for creation after all
other entries have been extracted.  Filenames inside the archive MUST NOT rely
upon the creation of a symlink before they are extracted: they must all be
"absolute" relative to the base.  Extractors MAY reject symbolic links
pointing outside of the extraction area by default.  Extractors SHOULD reject
ALL symbolic links if a leading path of any one entry is itself another
symbolic link: the keys of symbolic links themselves must be absolute and not
rely upon links.  Extractors may freely re-order symbolic links, or reject
them entirely.

The `owner` array, if present, must have at least one entry, the file owner.
This should be portable.  On Unix systems, a second entry in the array should
be interpreted as a group.  If the first entry in the array is `null` or an
empty string then the owner is unspecified and only the group is given.

Tools should reject owner arrays with more than two entries, absent a feature
to enable such.

Prefix should not be present if any of [`base64`, `jsonline`, `jsonmulti`] is
present.

Prefix must be non-empty if present; it must not start with `{`; it defaults
to `"X"`.  Note that content can start with `{` if either the `jsonline` or
`jsonmulti` bool has been set true.  As long as the entry's "type" has not
been set to non-file, these should be written out as-is.

Prejudicial paranoid suspicion should be applied when seeing an Extended
Attribute (`xattr`) in any namespace other than `user`.  All extended
attributes are in `namespace.attribute` form, so the `user.` prefix must
always be present.

If owners or ACLs of any kind are used, an extractor SHOULD consider a
two-pass extraction: on the first pass, create files accessible only to the
user performing the extraction (or a dummy user), so that permission
restrictions do not risk causing the extraction of a later file to error out
(eg, directory made read-only to user); the second pass can then apply any
fix-ups needed to make ownership and ACLs take effect.

A type of `"skip"` allows for comments, or disabling entries.  These entries
should still have unique filenames, in the same namespace as all other
entries.

In a cryptographic context, `skip` entries may allow for freely mutating data
in an attack.  A tool handling textar files in such an environment MAY have
imposed its own rules on what is permissible and so `skip` entries might not
always be permissible.  Similarly for entries which contain MIME types in the
`type` field.

All JSON parsing SHOULD allow for trailing commas, to allow for fallible
humans.  Portable `textar` files will not rely upon this.

The top-level MIME type of `signature` overlaps with existing MIME types for
`application/pgp-signature`, `application/x-pkcs7-signature`, etc.  The intent
is that use of `signature` top-level indicates to textar processors that they
MAY use this entry to verify some other entry.  Such entries should typically
still be extracted as regular files too.

FIXME: how about a final entry with `{"filename":".","type":"signature/foo"}`
as a signature over "everything before this entry"?  Or allowing this one
filename to be repeated, to allow for appending and re-signing?

#### Signature MIME namespace

These MIME types are known to exist; no extractor is in any way obliged by
this pseudo-standard to support any of them, but users may have their own
requirements.

* `signature/openpgp4`: OpenPGP RFC 4880 signatures, ASCII-armored unless
  `base64` is set, in which case unarmored.  If you're using "OpenPGP",
  "GnuPG", "gpg" or the like, around the year 2020, then this is what you
  have.
* `signature/openpgp5`: reserved for the day when rfc4880bis becomes a
  standard.  Insert inspirational lyrics here.
* `signature/signify`: OpenBSD's signify(1) format (always plain text);
  compatible with minisign(1) signatures (unless pre-hashed, but that is for
  large files and so not aligned with textar usage, right?).


## Examples

File `foo.textar`:

```
{"format":"textar/1"}
{"filename":"foo"}
XThe first line of text.
XA second line.
X
XThis is the fourth line, the third line was empty.

{"filename":"bar","base64":true}
SWYgeW91IGNhbiBrZWVwIHlvdXIgaGVhZCB3aGVuIGFsbCBhYm91dCB5b3UKICAgIEFyZSBsb3Np
bmcgdGhlaXJzIGFuZCBibGFtaW5nIGl0IG9uIHlvdSwKSWYgeW91IGNhbiB0cnVzdCB5b3Vyc2Vs
ZiB3aGVuIGFsbCBtZW4gZG91YnQgeW91LAogICAgQnV0IG1ha2UgYWxsb3dhbmNlIGZvciB0aGVp
ciBkb3VidGluZyB0b287CklmIHlvdSBjYW4gd2FpdCBhbmQgbm90IGJlIHRpcmVkIGJ5IHdhaXRp
bmcsCiAgICBPciBiZWluZyBsaWVkIGFib3V0LCBkb27igJl0IGRlYWwgaW4gbGllcywKT3IgYmVp
bmcgaGF0ZWQsIGRvbuKAmXQgZ2l2ZSB3YXkgdG8gaGF0aW5nLAogICAgQW5kIHlldCBkb27igJl0
IGxvb2sgdG9vIGdvb2QsIG5vciB0YWxrIHRvbyB3aXNlOgo=

```

More complex examples TBD.


## Inspiration

I wanted to take some files which _have_ to be scattered about to interoperate
with other tools, and sanely archive them in one place.  Lots of small files
which work as a unit.  The archive was to be encrypted, but I realized I could
use the same thing for git storage (perhaps using `git-crypt` for encryption
there if needed).

I did not want binary files.  The content must be editable in a text-editor.
I remembered that the Plan9 `ar(6)` used plain-text headers, in a fixed
format.  This allowed the contents to be seen in a text-editor, but also
allowed tools to skip N octets to get to the next entry.

To skip content, but leave content clearly distinguishable from headers, I
remembered the shar(1) file-format.  I don't want to run arbitrary code in the
archive, but we can use the same line-prefixing trick.  And then glue in
base64 for the _occasional_ binary file where absolutely needed.

To support arbitrary edits and later extraction, I decided to keep lengths out
of the headers.  An optional final section can be an index, to be recreated as
needed.

The support for JSON came in when I considered symlinks.  And then I wanted to
be able to embed the output of `jq .` as a blob, unmodified, and pipe entries
through jq(1) freely.  The requirement of `jsonmulti` for all intermediate
lines to be indented is designed to allow this.

The support of MIME types as a file type is for a vague notion of cloud-init
or desktop MIME dispatch.  To avoid ambiguity or frivolous usage, this
specification is explicit that such entries should NOT be extracted as regular
files.
