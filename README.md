JSON for the [Fix programming language](https://github.com/tttmmmyyyy/fixlang): a document is read
into a tree, or read straight into the values a program wants without a tree standing in between.

# Reading a document into a tree

```
import Json::{Json, Object, Json::{as_array, as_number, as_object}, Object::find, read};

let document = *read(text);
let coordinates = document.as_object.find("coordinates").as_some.as_array;
let first_x = coordinates.@(0).as_object.find("x").as_some.as_number;
```

A value is held unboxed, so an array of values holds them where they stand.

An object's members are an `Object`, which is `Array (String, Json)`: a name and the value standing
under it, in the order the document gives them. `find` walks that order, so a document whose objects
are large and read often is one to build a `HashMap` from once.

# Writing a document

```
import Json::{Json, Json::{array, number, object, string}, write};

let document = object([
    ("name", string("a point")),
    ("at", array([number(1.5), number(-2.25)]))
]);
let text = write(document);   // {"name":"a point","at":[1.5,-2.25]}
```

A number is written in the shortest decimal form that reads back as itself, so reading the text
answers with the tree that was written.

# Reading without a tree

`Json.Decoder` reads what you want and leaves the rest of the document unread. `read_object` hands
you each member as it meets it, and carries the value you build from one member to the next.

```
module Main;

import Json.Decoder::{
    Decoder,
    Decoder::{decode, is_named, read_array, read_bool, read_number, read_object, read_text,
              read_text_span}
};
import Std::{Array, Bool, F64, IO, String, Array::{push_back, @}, IO::println,
             Monad::pure, Result::as_ok, ToString::to_string};

type Point = struct { x : F64, y : F64 };
type Shape = struct { name : String, points : Array Point, closed : Bool };

read_point : Decoder Point;
read_point = read_object(Point { x : 0.0, y : 0.0 }, |name, point|
    if *is_named(name, "x") { let v = *read_number; pure $ point.set_x(v) };
    if *is_named(name, "y") { let v = *read_number; pure $ point.set_y(v) };
    pure $ point
);

read_shape : Decoder Shape;
read_shape = read_object(Shape { name : "", points : [], closed : false }, |name, shape|
    if *is_named(name, "name") {
        let span = *read_text_span;
        let text = *read_text(span);
        pure $ shape.set_name(text)
    };
    if *is_named(name, "points") {
        let points = *read_array([], |points| let p = *read_point; pure $ points.push_back(p));
        pure $ shape.set_points(points)
    };
    if *is_named(name, "closed") { let b = *read_bool; pure $ shape.set_closed(b) };
    pure $ shape
);

main : IO ();
main = (
    let text = "{ \"name\" : \"tri\", \"points\" : [ {\"x\":1.0,\"y\":2.0} ], \"closed\" : true }";
    let shape = read_shape.decode(text).as_ok;
    println(shape.@points.@(0).@x.to_string)   // 1.0
);
```

Three things the reading does for you.

- **A member you say nothing about is passed over.** The last `pure $ shape` answers for every name
  the reading does not know, and the value standing there is skipped.
- **The members may come in any order**, and a name the document does not carry simply leaves your
  starting value where it was.
- **White space is never yours to skip.** Every reading takes the space in front of it.

`read_number` answers with an `F64`, `read_bool` with a `Bool`, and `read_text_span` with where the
text stands, which `read_text` turns into a `String` with its escapes resolved. `skip_value` passes
over a value you want to reach past deliberately, and `read_member`, `read_separator`, `took` and
`take` are there for a document whose shape the two combinators do not fit.

A reading never goes back, which is what a JSON document allows: the byte the cursor stands on says
what comes next.

# Numbers

Both ways stand on the one place where floating point is exact. A whole number below 2^53 and a
power of ten below 10^23 are each held exactly, so scaling one by the other rounds once, and one
rounding of the exact product is the nearest double to it. A number the digits cannot reach that
way is read and written by the runtime's own conversion instead.

The reading agrees with `strtod` and the writing with `snprintf("%.*f")` over every value the tests
put to them.
