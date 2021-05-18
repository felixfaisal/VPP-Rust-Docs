First 100 lines of code 

```rust 
#[derive(Clap, Debug, Clone, Serialize, Deserialize, EnumString, Display)]
enum OptParseType {
    File,
    Tree,
    ApiType,
    ApiMessage,
}
``` 
So we have an enum here OptParseType that is either a File, Tree, ApiType or ApiMessage 
Now the real problem here is that I do not know what each of these things specifically means 
- File 
-  Tree 
-  ApiType 
-  ApiMessage 

```rust
/// Ingest the VPP API JSON definition file and output the Rust code
#[clap(version = "0.1", author = "Andrew Yourtchenko <ayourtch@gmail.com>")]
#[derive(Clap, Debug, Clone, Serialize, Deserialize)]
struct Opts {
    /// Input file name
    #[clap(short, long)]
    in_file: String,

    /// output file name
    #[clap(short, long, default_value = "dummy.rs")]
    out_file: String,

    /// parse type for the operation: Tree, File, ApiMessage or ApiType
    #[clap(short, long, default_value = "File")]
    parse_type: OptParseType,

    /// Print message names
    #[clap(long)]
    print_message_names: bool,

    /// Generate the code
    #[clap(long)]
    generate_code: bool,

    /// A level of verbosity, and can be used multiple times
    #[clap(short, long, parse(from_occurrences))]
    verbose: i32,
}
``` 
So there's some comments here that can prove to be usefull, We're ingesting the API JSON files and outputing the generated rust code in probably some file, So we define a structure `Opts`, Previous enum was `OptParseTypes`, Clearly Opts is short for something but we'll figure it out as we go along. 

- `struct Opts {` - We define an Opts structure 
- `in_file:String,` - This is going to have the input file name 
- `out_file:String,` - Ouput file name 
- `parse_type:OptParseType` - parse type for the operation: Tree, File, ApiMessage or ApiType, **So something makes sense or not** 
- `generate_code:bool` - I don't understand why this is a bool, Is it like used to check if code is generated or not etc, We will findout 
- `verbose: i32` - I don't understand shit what this means 

```rust
#[derive(Debug, Clone)]
struct VppJsApiType {
    type_name: String,
    fields: Vec<VppJsApiMessageFieldDef>,
}

impl Serialize for VppJsApiType {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(1 + self.fields.len()))?;
        seq.serialize_element(&self.type_name)?;
        for e in &self.fields {
            seq.serialize_element(e)?;
        }
        seq.end()
    }
}
``` 
`struct VppJsApiType` - I believe this holds the value for an API types, and is stored in `type_name` but don't understand what `fields` means but it seems be initialized as a vector 
So we are creating custom serializer for the struct VppJsApiType, Here's the link to read more 
[Serde Serializer](https://serde.rs/impl-serialize.html)

```rust
use serde::de::{self, Deserializer, SeqAccess, Visitor};
use std::fmt;
```
So we want to implement a custom deserializer, For which we are importing Deserializer, SeqAccess and Visitor traits

```rust
struct VppJsApiTypeVisitor;

impl<'de> Visitor<'de> for VppJsApiTypeVisitor {
    type Value = VppJsApiType;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("struct VppJsApiType")
    }

    fn visit_seq<V>(self, mut seq: V) -> Result<VppJsApiType, V::Error>
    where
        V: SeqAccess<'de>,
    {
        let type_name: String = seq
            .next_element()?
            .ok_or_else(|| de::Error::invalid_length(0, &self))?;
        let mut fields: Vec<VppJsApiMessageFieldDef> = vec![];
        loop {
            let nxt = seq.next_element();
            log::debug!("Next: {:#?}", &nxt);
            if let Ok(Some(v)) = nxt {
                fields.push(v);
            } else {
                break;
            }
        }
        Ok(VppJsApiType { type_name, fields })
    }
}
```
A Visitor is instantiated by a Deserialize impl and passed to a Deserializer. The Deserializer then calls a method on the Visitor in order to construct the desired type.
This is created to deseralize data into VppJsApiType form which is a struct 
You can read more about custom serde deserializer [here](https://serde.rs/impl-deserialize.html)



