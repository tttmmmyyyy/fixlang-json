# Json

Defined in json@0.10.0

A JSON document: the value it holds, the text it is written in, and the reading and writing that
carry one into the other.

`read` turns a text into a `Json`, whose members `Object::find` looks up by name, and `write`
turns a `Json` built from `object`, `array`, `number` and the other variants back into text.

```fix
let document = read("{\"name\":\"a point\",\"at\":[1.5,-2.25]}").as_ok;
let at = document.as_object.find("at").as_some.as_array;
assert_eq(|_|"", at.@(1).as_number, -2.25);;

let point = object([("x", number(1.5)), ("y", number(-2.25))]);
assert_eq(|_|"", write(point), "{\"x\":1.5,\"y\":-2.25}")
```

## Values

### namespace Json

#### read

Type: `Std::String -> Std::Result Std::ErrMsg Json::Json`

Reads a document, and answers with the value it holds.

The text is read as the bytes it is written in. A document the grammar does not accept answers
with a message naming what was expected and the byte it stood at; so does one that carries text
behind its value.

##### Parameters

* `text` - The document to read.

#### write

Type: `Json::Json -> Std::String`

Writes a document. Every number is written in the shortest decimal form that reads back as
itself, so reading the text answers with the tree that was written.

##### Parameters

* `document` - The tree to write.

### namespace Json::Object

#### find

Type: `Std::String -> Json::Object -> Std::Option Json::Json`

The value the member named `name` holds, or `none` when the object carries no such member.

##### Parameters

* `name` - The name to look for.
* `members` - The members of an object.

##### Examples

```fix
let members = read("{\"x\":1.5,\"y\":2.5}").as_ok.as_object;
assert_eq(|_|"", members.find("x"), some(number(1.5)));;
assert(|_|"", members.find("z").is_none)
```

## Types and aliases

### namespace Json

#### Json

Defined as: `type Json = unbox union { ...variants... }`

A JSON value.

The members of an object stand in the order the document gives them, and writing the value out
writes them in that order.

##### variant `null`

Type: `()`

##### variant `bool`

Type: `Std::Bool`

##### variant `number`

Type: `Std::F64`

##### variant `string`

Type: `Std::String`

##### variant `object`

Type: `Json::Object`

##### variant `array`

Type: `Std::Array Json::Json`

#### Object

Defined as: `type Object = Std::Array (Std::String, Json::Json)`

The members of an object: a name and the value standing under it, in the order the document
gives them.

## Traits and aliases

## Trait implementations

### impl `Json::Json : Std::Eq`

Two values are equal when they are the same kind and hold the same thing. Two objects are equal
when their members stand in the same order and hold the same values, so a document that names its
members in another order gives another value.

### impl `Json::Json : Std::Format`

Writes a value as JSON text, the same text `write` returns for it. A document can then be one
of the values `format` and `printf` fill a template with:
`"{{\"points\":{}}}".format((points,))`.