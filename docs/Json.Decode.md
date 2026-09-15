# Json.Decode

Defined in json@0.5.0

Reads what you want out of a JSON document and leaves the rest of it unread.

A `Decoder a` reads an `a` from where the cursor stands and leaves the cursor behind what it
read. Build one out of the primitives below, chain them with `*` and `;;`, and run it over a
document with `decode`.

```
// The number the member `x` of the object at the cursor holds.
_x : Decoder F64;
_x = (
    enter_object;;
    let span = *read_member;
    let is_x = *is_named(span, "x");
    if !is_x { Decoder::fail_expecting("the member x") };
    read_number
);

let value = *_x.decode("{\"x\": 1.5}");   // ok(1.5)
```

A string comes back as a `Span`, where its bytes stand in the document, and those bytes are the
ones the document holds: an escape such as `\n` or `\u0041` stands there as the escape.
`is_named` and `read_text` resolve the escapes; `is_named_as_written` reads the bytes as they
stand.

A reading that fails answers with a message naming what the grammar expected and where it stood.

## Values

### namespace Json.Decode

#### make_cursor

Type: `Std::String -> Json.Decode::Cursor`

A cursor standing at the first byte of a document.

##### Parameters

* `text` - The document to read.

#### make_span

Type: `Std::I64 -> Std::I64 -> Std::Bool -> Json.Decode::Span`

The span of the bytes from `from` up to `to`, which `escaped` says whether an escape stands
among.

##### Parameters

* `from` - The position of the first byte of the text.
* `to` - The position one past its last byte.
* `escaped` - Whether a backslash stands between the two.

### namespace Json.Decode::Decoder

#### advance

Type: `Json.Decode::Decoder ()`

Moves the cursor one byte on.

#### decode

Type: `Std::String -> Json.Decode::Decoder a -> Std::Result Std::ErrMsg a`

Runs a reading over a whole document, and answers with what it read.

##### Parameters

* `text` - The document to read.
* `decoder` - The reading to run.

##### Examples

```
read_number.decode("1.5")   // ok(1.5)
read_number.decode("x")     // err("expected a number at 0")
```

#### enter_array

Type: `Json.Decode::Decoder ()`

Takes the bracket an array opens with, leaving the cursor on its first element.

#### enter_object

Type: `Json.Decode::Decoder ()`

Takes the brace an object opens with, leaving the cursor on its first member.

#### fail_expecting

Type: `Std::String -> Json.Decode::Decoder a`

A reading that fails, saying what the grammar expected and where.

##### Parameters

* `what` - What the grammar expected there.

#### is_at

Type: `Std::U8 -> Json.Decode::Decoder Std::Bool`

Whether the byte at the cursor is `byte`, which is false where the document has ended.

##### Parameters

* `byte` - The byte to look for.

#### is_at_end

Type: `Json.Decode::Decoder Std::Bool`

Whether the cursor has reached the end of the document.

#### is_named

Type: `Json.Decode::Span -> Std::String -> Json.Decode::Decoder Std::Bool`

Whether the text of `span` is `name`.

A span that carries an escape has it resolved first, so a name written `"\\u0078"` answers to
`"x"`.

##### Parameters

* `span` - Where the name read from the document stands.
* `name` - The name to compare it against.

#### is_named_as_written

Type: `Json.Decode::Span -> Std::String -> Json.Decode::Decoder Std::Bool`

Whether the bytes `span` stands on are `name`, taken as the document writes them.

An escape among them stands as the escape, so a name written `"\\u0078"` does not answer to
`"x"`. Use this where the document is known to write its names plainly.

##### Parameters

* `span` - Where the name read from the document stands.
* `name` - The name to compare it against.

#### read_array

Type: `s -> (s -> Json.Decode::Decoder s) -> Json.Decode::Decoder s`

Reads the array the cursor stands on, handing each element to `element` and carrying the
state it answers with to the next one.

An element it reads nothing of is passed over here, as a member is in `read_object`.

##### Parameters

* `state` - What the reading starts from.
* `element` - What to do with one element.

##### Examples

```
read_numbers : Decoder (Array F64);
read_numbers = read_array([], |values| let v = *read_number; pure $ values.push_back(v));

read_numbers.decode("[1.0, 2.0, 3.0]")   // ok([1.0, 2.0, 3.0])
read_numbers.decode("[]")                // ok([])
```

#### read_bool

Type: `Json.Decode::Decoder Std::Bool`

Reads the `true` or `false` the cursor stands on.

#### read_member

Type: `Json.Decode::Decoder Json.Decode::Span`

Reads a member's name and the colon behind it, and answers with where the name stands.

#### read_null

Type: `Json.Decode::Decoder ()`

Reads the `null` the cursor stands on.

#### read_number

Type: `Json.Decode::Decoder Std::F64`

Reads a number, which runs to the first byte that no number can carry.

#### read_object

Type: `s -> (Json.Decode::Span -> s -> Json.Decode::Decoder s) -> Json.Decode::Decoder s`

Reads the object the cursor stands on, handing each member to `member` and carrying the
state it answers with to the next one.

`member` is handed where the name stands and the state so far. **A member it reads nothing
of is passed over here**, so a reading answers for the names it knows and leaves the rest
alone.

##### Parameters

* `state` - What the reading starts from.
* `member` - What to do with one member.

##### Examples

```
type Point = struct { x : F64, y : F64 };

read_point : Decoder Point;
read_point = read_object(Point { x : 0.0, y : 0.0 }, |name, point|
    if *is_named(name, "x") { let v = *read_number; pure $ point.set_x(v) };
    if *is_named(name, "y") { let v = *read_number; pure $ point.set_y(v) };
    pure $ point
);

read_point.decode("{ \"y\" : 2.0, \"z\" : 9.0, \"x\" : 1.0 }")
// ok(Point { x : 1.0, y : 2.0 })  -- any order, and `z` is passed over
```

#### read_separator

Type: `Std::U8 -> Json.Decode::Decoder Std::Bool`

Reads the byte that closes a member list or separates it from the next member, and answers
with whether another member follows.

##### Parameters

* `closing` - The byte that closes this list.

#### read_text

Type: `Json.Decode::Span -> Json.Decode::Decoder Std::String`

The text the span stands for, with every escape resolved.

##### Parameters

* `span` - Where the text stands.

##### Examples

```
// The text of a string value.
read_string : Decoder String;
read_string = (let span = *read_text_span; read_text(span));

read_string.decode("\"a b\"")        // ok("a b")
read_string.decode("\"\\u0078\"")     // ok("x")   -- the escape is resolved here

// The name of a member, which `read_member` hands you as a span.
read_name : Decoder String;
read_name = (enter_object;; let name = *read_member; read_text(name));

read_name.decode("{\"x\":1.5}")      // ok("x")
```

#### read_text_span

Type: `Json.Decode::Decoder Json.Decode::Span`

Reads the string the cursor stands on, and answers with where its text stands.

The walk to the closing quote sees every escape on its way, so the span it answers with says
whether one stands inside it.

#### run

Type: `Json.Decode::Cursor -> Json.Decode::Decoder a -> Std::Result Std::ErrMsg (a, Json.Decode::Cursor)`

Runs a reading over a cursor, and answers with what it read and the cursor behind it.

##### Parameters

* `cursor` - Where the reading begins.
* `decoder` - The reading to run.

#### skip_value

Type: `Json.Decode::Decoder ()`

Moves the cursor past the value it stands on, building nothing for it.

#### take

Type: `Std::U8 -> Std::String -> Json.Decode::Decoder ()`

Takes the byte `byte`, failing where the document holds something else.

##### Parameters

* `byte` - The byte the grammar asks for there.
* `what` - What to call it in the report.

#### took

Type: `Std::U8 -> Json.Decode::Decoder Std::Bool`

Takes the byte `byte` where the cursor stands on it, and answers whether it did. Where the
cursor stands on another byte, or the document has ended, it stays where it is.

##### Parameters

* `byte` - The byte to take.

#### took_null

Type: `Json.Decode::Decoder Std::Bool`

Takes the `null` where the cursor stands on it, and answers whether it did. Where the cursor
stands on anything else, it stays where it is, so that a member whose value may be `null` is
read by asking here first.

### namespace Json.Decode::Span

#### escaped

Type: `Json.Decode::Span -> Std::Bool`

Whether a backslash stands among the bytes, so that the text needs unescaping to be read.

#### from

Type: `Json.Decode::Span -> Std::I64`

The position of the first byte of the text.

#### to

Type: `Json.Decode::Span -> Std::I64`

The position one past the last byte of the text.

## Types and aliases

### namespace Json.Decode

#### Cursor

Defined as: `type Cursor = unbox struct { ...fields... }`

A document and how far into it a reading has come.

#### Decoder

Defined as: `type Decoder a = unbox struct { ...fields... }`

A reading that takes a value out of a document and leaves the position after it.

#### Span

Defined as: `type Span = unbox struct { ...fields... }`

Where the text of a string stands in a document, and whether an escape stands among its bytes.

## Traits and aliases

## Trait implementations

### impl `Json.Decode::Decoder : Std::Functor`

### impl `Json.Decode::Decoder : Std::Monad`