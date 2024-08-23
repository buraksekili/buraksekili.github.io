---
editPost:
  URL: "https://github.com/buraksekili/buraksekili.github.io/blob/main/content/articles/rust-serde.md"
  Text: "Edit this page on "
author: "Burak Sekili"
title: "Writing Custom Data Format in Rust using serde"
date: "2024-08-23"
description: "How to serialize and deserialize a custom custom data format in Rust using serde"
tags: ["Rust", "serde"]
TocOpen: true
---

# Writing Custom Data Format in Rust using serde

If you need to perform serialization or deserialization in Rust, you’ve most likely used the _serde_ before.
I’m currently learning Rust, and I found myself needing similar thing.
To get familiar with the Rust ecosystem, I decided to develop a simple key-value store.
Initially, the engine for this key-value store was designed to work with JSON objects, as JSON is a widely-used format that’s straightforward to use with web clients.
Also, as most of the languages and platforms already support JSON, it is a good choice to start with.

What I wanted to achieve was that users send JSON objects to indicate their request,
such as "give me the value of key <key>", which can be represented as `{ "key": "some_key" }` in JSON.
The response is also formatted as a JSON object, like `{"value": "value_of_the_key"}`. The concept and requirements of this server are quite simple and clear.

I generally use Go for my projects, where the JSON serialization/deserialization process is straightforward.
You can manage it with some tags on your struct, and the rest is handled by Go itself.
In Rust, achieving the same thing is also straightforward with _serde_.
You only need to add a few macros, and serde will automatically implement serialization and deserialization methods for you.
As JSON is one of the most popular data formats, you can set this up with just a few lines of code. I am not going to give details about how to achieve this as
[serde documentation](https://serde.rs/) has handful examples regarding this.

After checking similar projects, I realized that most of the engines do not use these widely adopted data formats in place to interact with underlying storage engine.
Of course, the reasons of this may vary but the fundamental concerns are more or less the same: performance and simplicity. In my case, users only `get`, `set` or `rm`
the key. So, using JSON is a bit overkill here.
I totally agree that it is quite straightforward to implement and get start with but what happens if I want to introduce a simple - and to be honest useless and dumbest -
data format to interact with my key-value storage engine? Considering that my learning purpose of Rust, the idea looks okay to me.
Then, i started to focus on how to work with custom data format with _serde_? So that I can serialize Rust data types to my data format and deserialize some byte sequence such
as strings back to Rust types?

## Concepts

Before jumping into implementation details, it'd be better to get familiar with some core concepts that we are going to be using.

### Data Format

First things first, let’s try to understand what the data format is. Simply put, a data format defines how the data is stored and structured.
For example, well-known data formats such as JSON, CSV, and TOML can store the same data but in different ways.
You probably use these formats on a daily basis as they are widely adopted and used. Almost every language support these data formats more or less.
Although these formats use different syntax while representing the data, the underlying data they represent is the same.

In my case, I wanted to introduce a simple custom data format, which looks like this:

```
\r:<cmd_name> <key> <optional_value>:
```

This format allows me to store a sequence of commands - essentially the requests that users send to the key-value engine, such as fetching a key - in a log file.
This format is of course not complicated and maybe even not quite helpful compared to JSON but it is of course cleaner and easier for me to work with.
For instance, using this format makes it easy for my parser to determine where a command request
starts in the file, as each command is prefixed with `\r:` and ends with a colon `:`.

This data format of course is NOT a JSON (or YAML, CSV and whatever) which means I cannot use existing deserializer of already knowns data formats.
This means I need to implement my own basic serializer and deserializer to convert Rust data structures,
such as enum or struct, into this custom format and vice versa.

> For simple use cases like mine, you do not even need to write a full `Serializer` and `Deserializer` using serde.
> You can implement a custom deserialize method and a visitor to handle most tasks. My primary goal in creating these custom serializers and deserializers is to learn and understand the process better.

### Data Type

The data types refer to the way data is stored in memory and classified in the language that we use.
In Rust and in many other languages, data types can be simple types such as integers, floating number, booleans and
composite ones like structs, enums, and classes.

On the other hand, data format is how this data type is stored and structured in the storage.

For example, the following `Get` struct in Rust corresponds to the data type:

```rust
struct Get {
    key: String,
}
```

where, storing this data type as a JSON string as `{"key": "some_key"}` is the data format.

### Serialization and Deserialization

With that being said, serialization is the process of converting or transforming these data types into a specific data format.
For example, if you use a JSON serializer, it will convert your Rust data types into a JSON compatible string representation.

The deserialization is kind of the reverse process of the serialization where it takes your input (like as a string, byte sequence or in binary)
and converts this input back into the data types, which might be struct or vector etc.

## Serialization

As clearly mentioned in [serde documentation](https://serde.rs/data-format.html), serde is NOT a parsing library. So, i am not going to dive into
how to parse the stream.

In our case, the request that users send is represented as enum type, as follows:

```rust
pub enum Request {
    Get { key: String },
    Set { key: String, val: String },
    Rm { key: String },
}
```

First start with serializing this data type into our custom data format:

```
\r:<cmd_name> <key> <optional_value>:
```

In order to serialize our enum, we need to implement [Serialize](https://docs.rs/serde/1.0.208/serde/ser/trait.Serialize.html) trait for our `Request`
type. This trait only has one required method called `serialize`. For most of the time, you do not need to implement the trait from scratch.
There is a helper procedural macro called `serde_derive` to implement the trait for you. So, let's add this macro to our type.

```rust
use serde::Serialize;

#[derive(Serialize)]
pub enum Request {
    Get { key: String },
    Set { key: String, val: String },
    Rm { key: String },
}
```

Eventually, this macro generates the following code (which shows the generated code in simplified manner) for `Request` struct.

```rust
// for more, please check https://docs.rs/serde/1.0.208/serde/ser/trait.Serialize.html
impl Serialize for Request {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        match *self {
            Request::Get { ref key } => {
                let mut s = serializer.serialize_struct_variant("Request", 0, "Get", 1)?;
                s.serialize_field("key", key)?;
                s.end()
            }
            Request::Set { ref key, ref val } => {
                let mut s = serializer.serialize_struct_variant("Request", 1, "Set", 2)?;
                s.serialize_field("key", key)?;
                s.serialize_field("val", val)?;
                s.end()
            }
            Request::Rm { ref key } => {
                let mut s = s.serialize_struct_variant("Request", 2, "Rm", 1)?;
                s.serialize_field("key", key)?;
                s.end(s)
            }
        }
    }
}
```

Instead of manually running `serialize_field` on each field of our enum, the macro will automatically generate the necessary code for us.
This makes it easier to use in most cases.

To understand the flow, let’s break down what the `serialize` method does. It knows how to instruct `Serializer` to serialize the `Request` struct.
Then, for each field in the enum, it calls `serialize_field` by using Serializer that is passed to `serialize` method.

If you want to use an existing serializer, like a JSON serializer, you can pass that serializer into the serialize method.
The serializer will then handle the serialization process for you, converting your Rust data types into a JSON compatible format.

However, in our case, we want to serialize the Request data type into our custom data format, which is not compatible with JSON or
any other standard data formats. This means we need to implement our own custom serialization logic to handle this specific format - at least as
scope of this blog post :).

To write our own Serializer, we need to implement [`serde::ser::Serializer` trait](https://docs.rs/serde/1.0.208/serde/trait.Serializer.html).
If you check the trait, it includes lots of required methods. But of course, not all of these methods need to be implemented in every case.
Most of the methods are related to serializing specific types, like structs or i32.

In our case, we need to serialize a struct type. Based on the code generated by Serialize macro, we call `serialize_struct_variant` method.
Therefore, we'll definitely need to implement this method.

Also, we need to specify some required associated types of Serializer trait: `Ok`, `Error` and `SerializeStructVariant` in the Serializer trait.

- `type Ok` corresponds to the output type that we generate after serialization.

In our case, we can use `()` for Ok value since we are going to store the the serialization result in-memory and then write it to `io::Write`.

- `type Error` corresponds to the error type that we may face during serialization.
  For the error type, we can define a custom `Error` type by following conventions [here]().

- `type SerializeStructVariant` corresponds to the type returned from the `serialize_struct_variant` method.
  We can set `SerializeStructVariant` to `Self`, meaning our `Serializer` will be returned as the result of the `serialize_struct_variant` method.
  This allows us to use the serialization methods that we define within our custom `Serializer`.

Here’s a simplified version of what our Serializer implementation will look like (omitting other methods that we don't need to implement):

```rust
impl<'a> ser::Serializer for &'a mut KvRequestSerializer {
    type Ok = ();
    type Error = Error;
    type SerializeStructVariant = Self;

    fn serialize_struct_variant(
        self,
        _name: &'static str,
        _variant_index: u32,
        variant: &'static str,
        _len: usize,
    ) -> std::result::Result<Self::SerializeStructVariant, Self::Error> {
        let req_type = match variant {
            "Set" => Ok("set"),
            "Get" => Ok("get"),
            "Rm" => Ok("rm"),
            _ => Err(Error::InvalidData(String::from("invalid request provided"))),
        }?;
        self.output += "\r";
        self.output += ":";
        self.output += req_type;
        Ok(self)
    }

    // and other required methods and types here with no implementation - you just need to define them.
}

impl<'a> ser::SerializeStructVariant for &'a mut KvRequestSerializer {
    type Ok = ();
    type Error = Error;

    fn serialize_field<T>(
        &mut self,
        _key: &'static str,
        value: &T,
    ) -> std::result::Result<(), Self::Error>
    where
        T: ?Sized + Serialize,
    {
        value.serialize(&mut **self)?;
        Ok(())
    }

    fn end(self) -> std::result::Result<Self::Ok, Self::Error> {
        self.output += ":";
        Ok(())
    }
}
```

So, for example in order to serialize `Request::Get {key: "abc"}`, based on the `serialize` method (which is generated by `Serialize` macro)

```rust
// serialize_struct_variant returns std::result::Result<Self::SerializeStructVariant, Self::Error> which means succesfull results
// yield Self::SerializeStructVariant. In our case, we defined `type SerializeStructVariant` as Self again. So, the result of
// `serialize_struct_variant` method will be `KvRequestSerializer`. So, in the following code, `s` is type of KvRequestSerializer
// which implements SerializeStructVariant trait. So that we can serialize the structs.
let mut s = serializer.serialize_struct_variant("Request", 0, "Get", 1)?;
// now, s points to KvRequestSerializer and we already implement SerializeStructVariant trait.
s.serialize_field("key", key)?;
s.end()
```

This flow will be resulted through following process:

1. `KvRequestSerializer`'s `serialize_struct_variant` method.
   Here, we have information about how our Rust data type look like.
2. `KvRequestSerializer`'s `serialize_field` method in `SerializeStructVariant` trait implementation.
   Here, we know each key of the Request::Get enum and its values
3. `KvRequestSerializer`'s `serialize_str`.
   Then we pass the value of `key` field to `serialize_str`. So that we can form our desired data format
4. Lastly we call `end` method from `SerializeStructVariant`

During the process, we store the result of each operation in the `output` field of our Serializer.
At the end, we can pass this `output` to our desired output location.

Now, we implemented our serializer. Let's write a simple test to verify the result. Before jumping into test cases, let's define
a function that eases usage of our serializer.

```rust
pub fn serialize<T: ser::Serialize>(request: &T) -> String {
    let mut serializer = KvRequestSerializer {
        output: String::new(),
    };

    request.serialize(&mut serializer).expect("failed to serialize");
    serializer.output
}

#[test]
fn test_serialization_request_struct() {
    use crate::request::Request;

    let get_request = Request::Get {
        key: "get_key_testing".to_owned(),
    };

    let expected_get = "\r:get get_key_testing:";
    assert_eq!(serialize(&get_request), expected_get);
}
```

Now that our serializer works as expected, let's move on to deserializing our custom data format into the `Request` type by implementing our own deserializer.

## Deserialization

Writing a custom deserializer can be more complicated than writing a serializer. I found myself confused quite often while working on the implementation at first. However, I'll try to simplify the explanation as much as possible.

In `serde`, deserialization is a two-phase process that involves both a `Deserializer` and a `Visitor`. Let's start with the `Deserializer`.

Deserializer is responsible for interpreting the input, which is a data format in form of string, byte, binary etc., and matching
this input data format to serde data model, such as integer, sequence and so on.
Here, the `serde` data model refers to the data types defined within `serde`, which correspond closely to Rust's type system.
For example, `bool` in `serde` corresponds to the `boolean` type in Rust.
The `serde` documentation provides a clear explanation of the data model, including an example using `OsString`, which is highly recommended
if you're not already familiar with `serde`'s data model. Please refer to the docs https://serde.rs/data-model.html.

Once the `Deserializer` matches input data to the appropriate `serde` data model,
a _Visitor_ is then used to analyze this generic data and convert it into the specific data type we want to achieve.

Although this process may sound complicated, let's break it down using a concrete example based on our case.
Suppose we have a string like `get abc:`. Our `deserialize` method will call `deserialize_str` on our custom deserializer.
This flow is similar to how the `Serializer` calls `serialize_struct_variant`, knowing that the data type is a struct.
In our case, we know that our data format is a string that contains a single request. Thus, we will call `deserialize_str`.
As opposed to `serialize` method, now we do not need to use macro to autogenerate `deserialize` method for `Request` type.
As we already know that the input data format will be a string, we will implement `deserialize` method in a way that it will
call `deserialize_str` method of our custom deserializer.

When the deserializer's `deserialize_str` method is called, it will, in turn, call the `visit_str` method on a visitor that we provide.
This means we need to implement our own Visitor to handle string representation of our custom data format.
Finally, within the `visit_str` method of our Visitor, we will parse the string and create the corresponding `Request` enum type based on the input.

Let's try to implement this deserialization process. As we did with the Serializer, we'll start with the deserialize method for
our data type - `Request` enum - which will instruct our custom deserializer.

```rust
impl<'de> de::Deserialize<'de> for Request {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: de::Deserializer<'de>,
    {
        deserializer.deserialize_str(RequestVisitor)
    }
}
```

If we look at the `deserialize` method, you'll notice that we pass our deserializer to this method
(similar to how we did it with the `serialize` method).
Since we know our custom data format is a string, we'll directly call `deserialize_str`.
This is actually suitable for our simple data format.
However, we also need to pass a `RequestVisitor` as an argument to `deserialize_str`.
This requires us to create a visitor that implements the [`de::Visitor` trait](https://docs.rs/serde/1.0.208/serde/de/trait.Visitor.html). The `RequestVisitor` will process the string input and try to generate the appropriate `Request` enum.

```rust
pub struct RequestVisitor;

impl<'de> de::Visitor<'de> for RequestVisitor {
    type Value = Request;

    fn expecting(&self, formatter: &mut std::fmt::Formatter) -> std::fmt::Result {
        formatter.write_str("a command in the format ':<cmd> <required_key> <optional_value>'")
    }

    fn visit_str<E>(self, cmd_str: &str) -> std::result::Result<Self::Value, E>
    where
        E: de::Error,
    {
        let inputs = cmd_str.split_whitespace().collect::<Vec<&str>>();
        let len = inputs.len();
        if len != 2 && len != 3 {
            return Err(de::Error::custom(
                "invalid command request provided, valid commands are 'get', 'set' and 'rm'",
            ));
        }

        let cmd_name = inputs[0];

        let get_key = |key: &str| -> String {
            let trimmed = key.trim();

            if trimmed.ends_with(":") {
                return trimmed
                    .get(0..trimmed.len() - 1)
                    .unwrap_or(trimmed)
                    .to_owned();
            }

            return trimmed.to_owned();
        };

        let key = inputs[1];

        match cmd_name {
            "get" => Ok(Request::Get { key: get_key(key) }),
            "rm" => Ok(Request::Rm { key: get_key(key) }),
            "set" => {
                let val = inputs[2].trim();

                Ok(Request::Set {
                    key: key.to_owned(),
                    val: get_key(val),
                })
            }
            _ => Err(de::Error::custom(
                "invalid command is provided, valid commands are 'get', 'set' and 'rm'",
            )),
        }
    }
}
```

The `Visitor` trait has only one required method: `expecting` which will be used in error messages.
While other methods like `visit_str` have default implementations, we need to override and implement the methods necessary for our use case.
In this case, since we are working with string values, we'll implement `visit_str`, which will handle parsing the given string.

For our example, `visit_str` will expect strings like `get abc:`, `set key value:`, or `rm key:`.
The `visit_str` method will attempt to convert these strings into the corresponding `Request` enum variants.

Finally we can implement our deserializer which will simply call the visitor's `visit_str` method, as follows:

```rust
pub struct Deserializer<'de> {
    input: &'de str,
}

impl<'de, 'a> de::Deserializer<'de> for &'a mut Deserializer<'de> {
    type Error = Error;
    fn deserialize_str<V>(self, visitor: V) -> std::result::Result<V::Value, Self::Error>
    where
        V: de::Visitor<'de>,
    {
        visitor.visit_str::<Self::Error>(&self.input)
    }

    // rest of the deserialize_* methods. as Serializer, i do not list them all here as there is a deserialize_*
    // method for almost all type.
    // For more detail about it, please refer to the:
    //      https://docs.rs/serde/1.0.208/serde/trait.Deserializer.html#
}
```

This approach keeps the deserialization process straightforward and clean. Now, let's write a simple test scenario to verify our implementation.

```rust
fn deserialize<'a, T: de::Deserialize<'a>>(input: &'a str) -> Result<T> {
    let mut deserializer = Deserializer::from_str(input);
    let t = T::deserialize(&mut deserializer)?;

    Ok(t)
}

#[test]
fn test_deserialize_set() {
    let data = r"set burak 123:";
    let expected = Request::Set {
        key: "burak".to_string(),
        val: "123".to_string(),
    };

    let result: Request = deserialize(data).expect("failed to deserialize");
}
```

---

I hope this post helps you get started with your own `serde` implementations.
Since I am also a beginner in Rust, I encourage you to always refer to the official `serde` documentation as the
primary source of truth.

Also, the source code is available on GitHub: https://github.com/buraksekili/kvs_protocol/

If you notice any mistakes or have feedback, feel free to reach out to me on Twitter, LinkedIn, or GitHub.

## References

- https://serde.rs/
- https://owengage.com/writing/2021-08-14-exploring-serdes-data-model-with-a-toy-deserializer/
