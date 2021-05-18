Second 100 lines of Code 

```rust
impl<'de> Deserialize<'de> for VppJsApiType {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_seq(VppJsApiTypeVisitor)
    }
}
```
We implement the custom deserealizer since we have already defined the Visitor, We don't have to write much here 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
enum VppJsApiDefaultValue {
    Str(String),
    Bool(bool),
    I64(i64),
    F64(f64),
}
```
So we are creating a new enum `VppJsApiDefaultValue`, It seems to have 4 types that is 
- String 
- Bool 
- i64
- F64 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiFieldOptions {
    #[serde(default)]
    default: Option<VppJsApiDefaultValue>,
}
```
Another struct with just one value `default:Option<VppJsApiDefaultValue>`m So we're creating a struct to hold the enum `VppJsApiDefaultValue` 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
enum VppJsApiFieldSize {
    Fixed(usize),
    Variable(Option<String>),
}
```
Another enum of FieldSize, So either it can have `Fixed(usize)` or `Variable(Option<String>)` 
```rust
#[derive(Debug, Clone)]
struct VppJsApiMessageFieldDef {
    ctype: String,
    name: String,
    maybe_size: Option<VppJsApiFieldSize>,
    maybe_options: Option<VppJsApiFieldOptions>,
}
```
We're defining the struct `VppJsApiMessageFieldDef` which consists  of 
- `ctype` - A String 
- `name` - A String 
- `maybe_size` - Is enum `AppJsApiFieldSize` which is wrapped under an option 
- `maybe_options` - is struct `VppJsApiFieldOptions` wrapped under an option 

```rust 
impl Serialize for VppJsApiMessageFieldDef {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        use crate::VppJsApiFieldSize::*;

        let mut len = 2;
        if self.maybe_options.is_some() {
            len = len + 1
        }
        len = len
            + match &self.maybe_size {
                None => 0,
                Some(Fixed(n)) => 1,
                Some(Variable(None)) => 1,
                Some(Variable(Some(x))) => 2,
            };
        let mut seq = serializer.serialize_seq(Some(len))?;
        seq.serialize_element(&self.ctype)?;
        seq.serialize_element(&self.name)?;
        match &self.maybe_size {
            None => { /* do nothing */ }
            Some(Fixed(n)) => {
                seq.serialize_element(&n);
            }
            Some(Variable(None)) => {
                seq.serialize_element(&0u32);
            }
            Some(Variable(Some(x))) => {
                seq.serialize_element(&0u32);
                seq.serialize_element(&x);
            }
        }

        if let Some(o) = &self.maybe_options {
            seq.serialize_element(o);
        }
        seq.end()
    }
}
```
We are implementing custom serializer for `VppJsApiMessageFieldDef` 
Next we are going to implement custom visitor for custom deserialization to take place 

```rust
struct VppJsApiMessageFieldDefVisitor;

impl<'de> Visitor<'de> for VppJsApiMessageFieldDefVisitor {
    type Value = VppJsApiMessageFieldDef;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("struct VppJsApiMessageField")
    }

    fn visit_seq<V>(self, mut seq: V) -> Result<VppJsApiMessageFieldDef, V::Error>
    where
        V: SeqAccess<'de>,
    {
        let ctype: String = if let Some(VppJsApiMessageFieldHelper::Str(s)) = seq.next_element()? {
            s
        } else {
            panic!("Error");
        };
        let name: String = if let Some(VppJsApiMessageFieldHelper::Str(s)) = seq.next_element()? {
            s
        } else {
            panic!("Error 2");
        };

        let mut maybe_sz: Option<usize> = None;
        let mut maybe_sz_name: Option<String> = None;
        let mut maybe_options: Option<VppJsApiFieldOptions> = None;

        loop {
            let nxt = seq.next_element();
            match nxt? {
                Some(VppJsApiMessageFieldHelper::Map(m)) => {
                    maybe_options = Some(m);
                    break;
                }
                Some(VppJsApiMessageFieldHelper::Str(o)) => {
                    maybe_sz_name = Some(o);
                }
                Some(VppJsApiMessageFieldHelper::Usize(o)) => {
                    maybe_sz = Some(o);
                }
                None => break,
            }
        }
        let maybe_size = match (maybe_sz, maybe_sz_name) {
            (None, None) => None,
            (Some(0), None) => Some(VppJsApiFieldSize::Variable(None)),
            (Some(0), Some(s)) => Some(VppJsApiFieldSize::Variable(Some(s))),
            (Some(x), None) => Some(VppJsApiFieldSize::Fixed(x)),
            (None, Some(s)) => panic!("Unexpected dependent field {} with no length", s),
            (Some(x), Some(s)) => panic!("Unexpected dependent field {} with length {}", s, x),
        };
        let ret = VppJsApiMessageFieldDef {
            ctype,
            name,
            maybe_size,
            maybe_options,
        };
        Ok(ret)
    }
}

impl<'de> Deserialize<'de> for VppJsApiMessageFieldDef {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_seq(VppJsApiMessageFieldDefVisitor)
    }
}
```
