# THRAY format specification

*Work in progress.*

(This is an informal specification of the THRAY format. In the future, an RFC may be planned.)

The THRAY format is a superset of JSON. The format extends JSON with support for comments and more data types.

Like JSON, THRAY is a text format, that is, a sequence of characters where each character is a Unicode code point.

## Conventions used in this document

The grammar syntax used in this specification is ABNF as defined in [RFC 5234](https://datatracker.ietf.org/doc/html/rfc5234) and [RFC 7405](https://datatracker.ietf.org/doc/html/rfc7405).

## Document

The text representing a valid THRAY format is called a THRAY document. A THRAY document contains a THRAY value with optional whitespace and comments.

```abnf
document = value-with-wsc

value = null /
    boolean /
    float /
    integer /
    string /
    binary /
    array /
    object /
    extension
```

## Comments and whitespace

Whitespace is defined to be space or tab characters. Newline (line separator) is defined to be either line feed (Unix style) or {carriage return, line feed} (Windows style) character sequences.

```abnf
newline = LF / CRLF
```

Comments are segments of text that carry no semantic meaning to the represented data. They are used for user documentation purposes and may be present wherever whitespace is permitted.

```abnf
wsc = *( 1*WSP / comment )
value-with-wsc = wsc value wsc
```

There are two types of comments: line comments and block comments.

```abnf
comment = line-comment / block-comment
```

### Line comments

Line comments begin with a character sequence of {slash, slash}, followed by zero or more of space, tab, printable ASCII, or Unicode characters.

```abnf
line-comment = "//" *( WSP / VCHAR / unicode-char )
```

### Block comments

Block comments are delimited by character sequences {slash, asterisk} and {asterisk, slash}. The content consists of zero or more of space, tab, carriage return, line feed, printable ASCII, and Unicode characters (excluding the {asterisk, slash} sequence as defined in the grammar).

```abnf
block-comment = "/*"
    *(
        block-comment-char-no-asterisk
        / ( "*" block-comment-char-no-slash )
    )
    "*/"
block-comment-char-no-asterisk = WSP / CR / LF / vchar-no-asterisk / unicode-char
block-comment-char-no-slash = WSP / CR / LF / vchar-no-slash / unicode-char
vchar-no-asterisk = %x21-29 / %x2B-7E
vchar-no-slash = %x21-2E / %x30-7E
```

## Null data type

The null data type can be used to represent an absence of a value. It has only one case-sensitive token.

```abnf
null = %s"null"
```

## Boolean data type

The boolean data type represents a boolean value. It has only two case-sensitive tokens.

```abnf
boolean = %s"true" / %s"false"
```

## Numbers

Unlike in JSON, numbers have two distinct data types: integer and float.

Numbers can be prefixed with optional character `+` or `-` to indicate sign.

Digits are sequence of one or characters for the appropriate base. Digits can be grouped using an underscore (`_`) to increase readability. The grouping character must be preceded and followed by a digit character.

```abnf
sign = "+" / "-"
digits-with-groups = 1*DIGIT *( "_" 1*DIGIT )
hex-digits-with-groups = 1*HEXDIG *( "_" 1*HEXDIG )
```

### Integer data type

The integer data type represents whole numbers. Integers can be expressed in decimal or hexadecimal notations.

A base 10 integer is a sequence of digits. The set of characters for digits range from "0" to "9" inclusive.

A base 16 (hexadecimal) integer is a sequence of `0x` (case-sensitive) followed by digits. The set of characters for digits range from "0" to "9" and "A" to "F" inclusive, case-insensitive.

```abnf
integer = [sign] ( hex-integer / decimal-integer )
decimal-integer = digits-with-groups
hex-integer = %s"0x" hex-digits-with-groups
```

Implementations must support number sizes up to 64-bit signed integer values. Implementations may support larger values.

### Float data type

The float data type represents IEEE 754 double-precision floating point numbers.

Floats are a sequence of digits, a decimal separator, digits, and an optional exponent part.

```abnf
float = [sign] ( infinity / nan / float-digits )
float-digits = digits-with-groups "." digits-with-groups [exponent-part]
exponent-part = "e" [sign] digits-with-groups
```

Other mathematical values, `Infinity` and `NaN`, are supported as case-sensitive tokens.

```abnf
infinity = %s"Infinity"
nan = %s"NaN"
```

## String data type

The string data type represents text values. Strings are delimited by a double quote character. The content consists of zero or more literals or escape sequences.

A string literal is a character interpreted without any transformations. These are space, ASCII printable characters excluding double quote and backslash, and Unicode characters.

```abnf
string = DQUOTE *string-fragment DQUOTE *string-line-continuation
string-fragment = string-literal / escape-sequence
string-literal = 1*(
    SP /
    %x21 / ; ASCII printable excluding double quote and backslash
    %x23-5B /
    %x5D-7E /
    unicode-char
)
backslash = %x5C
```

An escape sequence is a sequence of characters that represent an encoded character. An escape sequence begins with the escape character backslash followed by sequence of characters defined in the following table.

| Unescaped | Escaped |
| --------- | -------- |
| quotation mark (`"`) | `\"` |
| backslash (`\`) | `\\` |
| slash (`/`) | `\/` |
| backspace (U+0008) | `\b` |
| form feed (U+000C) | `\f` |
| line feed (U+000A) | `\n` |
| carriage return (U+000D) | `\r` |
| tab (U+0009) | `\t` |
| Unicode codepoint (BMP only) | `\uHHHH` where H is a hexadecimal character |
| Unicode codepoint | `\u{N}` where N is 1 to 6 hexadecimal characters |

```abnf
escape-sequence = backslash (
        DQUOTE /
        backslash /
        "b" / ; U+0008
        "f" / ; U+000C
        "n" / ; U+000A
        "r" / ; U+000D
        "t" / ; U+0009
        char-code-brace /
        char-code-bmp
    )
char-code-brace = %s"u{" 1*6HEXDIG "}"
char-code-bmp = %s"u" 4HEXDIG
```

### Surrogate pairs

When using the JSON 4-hexadecimal character Unicode escape sequence, only code points in the Basic Multilingual Plane may be used. To represent code points outside the plane, the character is represented using UTF-16 surrogate pairs.

Implementations must decode surrogate pairs into their corresponding characters. Implementations must return an error by default if decoding surrogate pairs fails. Implementations may use an alternate error handling, such as substitution by a Replacement Character (U+FFFD), if configured by the user.

Implementations should use the brace syntax for characters outside the BMP.

### Line continuation

Line continuations can be used to break long lines for readability purposes. A line continuation is notated by a string fragment, backslash, line separator, zero or more whitespace, and a string fragment.

```abnf
string-line-continuation = backslash newline *WSP DQUOTE *string-fragment DQUOTE
```

Introducing or removing line continuations must not affect the value of the string.

## Binary data type

The binary data type represents a sequence of bytes.

Binary values can be encoded in hexadecimal or base 64 notations.

```abnf
binary = binary-hex / binary-base64
```

Binary value sequences begin with a token representing the encoding, followed by the encoded value delimited by opening parenthesis and closing parenthesis characters.

```abnf
binary-hex = %s"b16(" *HEXDIG ")"
binary-base64 = %s"b64(" *( ALPHA / DIGIT / "_" / "-" ) ")"
```

Hexadecimal encoded binary values use the encoding token `b16`. Encoding is case-insensitive. However, implementations should use lowercase.

In base 64 encoding, the scheme "Base 64 Encoding with URL and Filename Safe Alphabet" defined in [RFC 4648](https://www.rfc-editor.org/rfc/rfc4648.html). Padding characters must not used. The encoding token is `b64`.

## Array data type

The array data type represents a list of values.

An array is a structure delimited by opening bracket and closing bracket characters. Values are separated by a comma. A trailing comma is optional.

```abnf
array = "[" ( array-items / array-empty ) "]"
array-empty = wsc
array-items = value-with-wsc *( "," value-with-wsc) [ "," wsc ]
```

## Object data type

The object data type represents a list of key-value pairs.

An object is a structure delimited by opening curly brace and closing curly brace characters. Key-value pairs are separated by a comma. A trailing comma is optional.

Key-value pairs are a sequence of a THRAY value, colon character, and a THARY value.

```abnf
object = "{" ( object-pairs / object-empty ) "}"
object-empty = wsc
object-pairs = object-pair * ( "," object-pair ) [ "," wsc ]
object-pair = value-with-wsc ":" value-with-wsc
```

Implementations may choose to present the object as a list or a map.

Because keys can be any THRAY value, it is important for implementations to carefully consider the conversion process to a map. Implementations may opt to only accept integer and string keys and return an error otherwise. Implementations must return an error if there are duplicate keys by default. Accepting duplicates may introduce security vulnerabilities when implementations use differing deduplication schemes.

## Extension data type

The extension data type annotates a value with a tag for use with custom data types. Extensions are delimited by a less-than sign and a greater-than sign characters. The content is a sequence of a tag, colon character, and a value.

Tags are one or more of alphanumerical, underscore, or hyphen characters.

```abnf
extension = "<" extension-tag ":" value-with-wsc ">"
extension-tag = 1*( ALPHA / DIGIT / "_" / "-" )
```

Applications should avoid tags consisting of 1 to 3 characters or simple words for private use. These tags are intended for use with well-known standards in subsequent specifications. Applications may opt to use a namespace prefix (such as `az-`) to avoid naming conflicts.

## Miscellaneous grammar rules

Miscellaneous grammar rules common to various data types:

```abnf
unicode-char = %x80-10FFFF
```

Rules not defined in this specification are Core ABNF rules defined in [Appendix B.1 in RFC 5234](https://datatracker.ietf.org/doc/html/rfc5234#appendix-B.1).

## Encoding

For data interchange, a THRAY document should be encoded as UTF-8. When using the UTF-8 encoding, a Byte Order Mark (U+FEFF) must not be used at the start of the document.

## File registration

For filesystems, the filename extension of `thray` is suggested to be used.

For MIME type, there is a possibility of registering `application/thray` but this is not guaranteed.
