Synopsis
========

`#include <ucl.h>`

Description
===========

Libucl is a parser and `C` API to parse and generate `ucl` objects. Libucl consist of several groups of functions:

### Parser functions
Used to parse `ucl` files and provide interface to extract `ucl` object

### Emitting functions
Convert `ucl` objects to some textual or binary representation.

### Conversion functions
Help to convert `ucl` objects to C types

### Generation functions
Allow creating of `ucl` objects from C types

### Iteration functions
Iterate over `ucl` objects

### Utility functions
Provide basic utilities to manage `ucl` objects

# Parser functions

Parser functions operates with `struct ucl_parser`.

### ucl_parser_new

~~~C
struct ucl_parser* ucl_parser_new (int flags);
~~~

Creates new parser with the specified flags:

- `UCL_PARSER_KEY_LOWERCASE` - lowercase keys parsed
- `UCL_PARSER_ZEROCOPY` - try to use zero-copy mode when reading files (in zero-copy mode text chunk being parsed without copying strings so it should exist till any object parsed is used)

### ucl_parser_register_macro

~~~C
void ucl_parser_register_macro (struct ucl_parser *parser,
    const char *macro, ucl_macro_handler handler, void* ud);
~~~

Register new macro with name .`macro` parsed by handler `handler` that accepts opaque data pointer `ud`. Macro handler should be of the following type:

~~~C
bool (*ucl_macro_handler) (const unsigned char *data,
    size_t len, void* ud);`
~~~

Handler function accepts macro text `data` of length `len` and the opaque pointer `ud`. If macro is parsed successfully the handler should return `true`. `false` indicates parsing failure and the parser can be terminated.

### ucl_parser_register_variable

~~~C
void ucl_parser_register_variable (struct ucl_parser *parser,
    const char *var, const char *value);
~~~

Register new variable $`var` that should be replaced by the parser to the `value` string.

### ucl_parser_add_chunk

~~~C
bool ucl_parser_add_chunk (struct ucl_parser *parser, 
    const unsigned char *data, size_t len);
~~~

Add new text chunk with `data` of length `len` to the parser. At the moment, `libucl` parser is not a streamlined parser and chunk *must* contain the *valid* ucl object. For example, this object should be valid:

~~~json
{ "var": "value" }
~~~

while this one won't be parsed correctly:

~~~json
{ "var": 
~~~

This limitation may possible be removed in future.

### ucl_parser_add_file

~~~C
bool ucl_parser_add_file (struct ucl_parser *parser, 
    const char *filename);
~~~

Load file `filename` and parse it with the specified `parser`. This function uses `mmap` call to load file, therefore, it should not be `shrinked` during parsing. Otherwise, `libucl` can cause memory corruption and terminate the calling application. This function is also used by the internal handler of `include` macro, hence, this macro has the same limitation.

### ucl_parser_get_object

~~~C
ucl_object_t* ucl_parser_get_object (struct ucl_parser *parser);
~~~

If the `ucl` data has been parsed correctly this function returns the top object for the parser. Otherwise, this function returns the `NULL` pointer. The reference count for `ucl` object returned is increased by one, therefore, a caller should decrease reference by using `ucl_object_unref` to free object after usage.

### ucl_parser_get_error

~~~C
const char *ucl_parser_get_error(struct ucl_parser *parser);
~~~

Returns the constant error string for the parser object. If no error occurred during parsing a `NULL` object is returned. A caller should not try to free or modify this string.

### ucl_parser_free

~~~C
void ucl_parser_free (struct ucl_parser *parser);
~~~

Frees memory occupied by the parser object. The reference count for top object is decreased as well, however if the function `ucl_parser_get_object` was called previously then the top object won't be freed.

### ucl_pubkey_add

~~~C
bool ucl_pubkey_add (struct ucl_parser *parser, 
    const unsigned char *key, size_t len);
~~~

This function adds a public key from text blob `key` of length `len` to the `parser` object. This public key should be in the `PEM` format and can be used by `.includes` macro for checking signatures of files included. `Openssl` support should be enabled to make this function working. If a key cannot be added (e.g. due to format error) or `openssl` was not linked to `libucl` then this function returns `false`.

### ucl_parser_set_filevars

~~~C
bool ucl_parser_set_filevars (struct ucl_parser *parser, 
    const char *filename, bool need_expand);
~~~

Add the standard file variables to the `parser` based on the `filename` specified:

- `$FILENAME` - a filename of `ucl` input
- `$CURDIR` - a current directory of the input

For example, if a `filename` param is `../something.conf` then the variables will have the following values:

- `$FILENAME` - "../something.conf"
- `$CURDIR` - ".."

if `need_expand` parameter is `true` then all relative paths are expanded using `realpath` call. In this example if `..` is `/etc/dir` then variables will have these values:

- `$FILENAME` - "/etc/something.conf"
- `$CURDIR` - "/etc"

## Parser usage example

The following example loads, parses and extracts `ucl` object from stdin using `libucl` parser functions (the length of input is limited to 8K):

~~~C
char inbuf[8192];
struct ucl_parser *parser = NULL;
int ret = 0, r = 0;
ucl_object_t *obj = NULL;
FILE *in;

in = stdin;
parser = ucl_parser_new (0);
while (!feof (in) && r < (int)sizeof (inbuf)) {
	r += fread (inbuf + r, 1, sizeof (inbuf) - r, in);
}
ucl_parser_add_chunk (parser, inbuf, r);
fclose (in);

if (ucl_parser_get_error (parser)) {
	printf ("Error occured: %s\n", ucl_parser_get_error (parser));
	ret = 1;
}
else {
    obj = ucl_parser_get_object (parser);
}

if (parser != NULL) {
	ucl_parser_free (parser);
}
if (obj != NULL) {
	ucl_object_unref (obj);
}
return ret;
~~~