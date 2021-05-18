Third 100 lines of code 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiMessageInfo {
    crc: String,
}

#[derive(Debug, Clone)]
struct VppJsApiMessage {
    name: String,
    fields: Vec<VppJsApiMessageFieldDef>,
    info: VppJsApiMessageInfo,
}
``` 
Now we're defining a struct `VppJsApiMessage`  which has `info: VppJsApiMessageInfo` that is another struct which has just one field that is `crc:String`. There is however a need for me to know having just one field with primitive type to be made into a struct, the reason behind it. 

```rust
impl Serialize for VppJsApiMessage {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(1 + self.fields.len() + 1))?;
        seq.serialize_element(&self.name)?;
        for e in &self.fields {
            seq.serialize_element(e)?;
        }
        seq.serialize_element(&self.info)?;
        seq.end()
    }
}
```
A serializer implemented for `VppJsApiMessage`, As you can see large part of the code is revolved around defining structs, enums and their respective serializers and deserializer. If you notice carefully you can see similar patterns emerge in the code with serializer with just few changes. It doesn't seem like redundant code but maybe something could be done better. 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
enum VppJsApiMessageHelper {
    Field(VppJsApiMessageFieldDef),
    Info(VppJsApiMessageInfo),
    Name(String),
}
```
We define another enum `VppJsApiMessageHelper` which has `VppJsApiMessageFieldDef` and `VppJsApiMessageInfo`. 
We are also creating custom visitor for `VppApiMessage` for desearlizing 

```rust
struct VppJsApiMessageVisitor;

impl<'de> Visitor<'de> for VppJsApiMessageVisitor {
    type Value = VppJsApiMessage;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("struct VppJsApiMessage")
    }

    fn visit_seq<V>(self, mut seq: V) -> Result<VppJsApiMessage, V::Error>
    where
        V: SeqAccess<'de>,
    {
        let name: String = if let Some(VppJsApiMessageHelper::Name(s)) = seq.next_element()? {
            s
        } else {
            panic!("Error");
        };
        log::debug!("API message: {}", &name);
        let mut fields: Vec<VppJsApiMessageFieldDef> = vec![];
        let mut maybe_info: Option<VppJsApiMessageInfo> = None;
        loop {
            let nxt = seq.next_element();
            log::debug!("Next: {:#?}", &nxt);
            match nxt? {
                Some(VppJsApiMessageHelper::Field(f)) => fields.push(f),
                Some(VppJsApiMessageHelper::Info(i)) => {
                    if maybe_info.is_some() {
                        panic!("Info is already set!");
                    }
                    maybe_info = Some(i);
                    break;
                }
                x => panic!("Unexpected element {:?}", x),
            }
        }
        let info = maybe_info.unwrap();
        Ok(VppJsApiMessage { name, fields, info })
    }
}

impl<'de> Deserialize<'de> for VppJsApiMessage {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_seq(VppJsApiMessageVisitor)
    }
}
```



