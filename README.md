JSON for the [Fix programming language](https://github.com/tttmmmyyyy/fixlang): a document is read
into a tree, or read straight into the values a program wants without a tree standing in between.

# Reading a document into a tree

```
import Json::{Json, Json::{as_array, as_number, as_object, find}, read};

let document = *Json::read(text);
let coordinates = document.as_object.Json::find("coordinates").as_some.as_array;
let first_x = coordinates.@(0).as_object.Json::find("x").as_some.as_number;
```

A value is held unboxed, so an array of values holds them where they stand. The members of an
object stand in the order the document gives them, and `find` scans them.

# Writing a document

```
import Json::{Json, Json::{array, number, object, string}, write};

let document = Json::object([
    ("name", Json::string("a point")),
    ("at", Json::array([Json::number(1.5), Json::number(-2.25)]))
]);
let text = Json::write(8_U8, document);   // {"name":"a point","at":[1.5,-2.25]}
```

`write` takes how many places after the point a number is written to. A number carrying further
than that loses the places beyond it.

# Reading without a tree

`Json.Decode` reads what you want and leaves the rest of the document unread.

```
import Json.Decode::Decoder::{decode, expect, member, name_bytes, named, number};

// The number the member `x` of the object at the cursor holds.
_x : Decoder F64;
_x = (
    expect('{', "an object");;
    let span = *member;
    let is_x = *named(span, name_bytes("x"));
    if !is_x { Decoder::expected("the member x") };
    number
);

let value = *_x.decode("{\"x\": 1.5}");   // ok(1.5)
```

A reading never goes back, which is what a JSON document allows: the byte the cursor stands on
says what comes next.

# Numbers

Both ways stand on the one place where floating point is exact. A whole number below 2^53 and a
power of ten below 10^23 are each held exactly, so scaling one by the other rounds once, and one
rounding of the exact product is the nearest double to it. A number the digits cannot reach that
way is read and written by the runtime's own conversion instead.

The reading agrees with `strtod` and the writing with `snprintf("%.*f")` over every value the tests
put to them.
