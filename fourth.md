Fourth 100 lines of code 

```rust
#[derive(Debug, Deserialize)]
struct VppJsApiAlias {
    #[serde(rename = "type")]
    ctype: String,
    length: Option<usize>,
}

impl Serialize for VppJsApiAlias {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut len = 1;
        if self.length.is_some() {
            len = len + 1;
        }
        let mut map = serializer.serialize_map(Some(len))?;
        map.serialize_entry("type", &self.ctype)?;
        if let Some(s) = &self.length {
            map.serialize_entry("length", s);
        }
        map.end()
    }
}
```
We are defining another struct `VppJsApiAlias` and respective serializer, However I'd like to know `#[serde(rename="type")]` means 

```rust
#[derive(Debug, Deserialize)]
struct VppJsApiService {
    #[serde(default)]
    events: Vec<String>,
    reply: String,
    stream: Option<bool>,
    #[serde(default)]
    stream_msg: Option<String>,
}

impl Serialize for VppJsApiService {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut len = 1;
        if self.stream.is_some() {
            len = len + 1;
        }
        if self.events.len() > 0 {
            len = len + 1;
        }
        if self.stream_msg.is_some() {
            len = len + 1;
        }
        let mut map = serializer.serialize_map(Some(len))?;
        if self.events.len() > 0 {
            map.serialize_entry("events", &self.events);
        }
        map.serialize_entry("reply", &self.reply)?;
        if let Some(s) = &self.stream {
            map.serialize_entry("stream", s);
        }
        if let Some(s) = &self.stream_msg {
            map.serialize_entry("stream_msg", s);
        }
        map.end()
    }
}
``` 
We have a new struct `VppJsApiService`, Some of the fields seem to make sense as I feel I have read them somewhere in the Vpp wiki. 

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiOptions {
    version: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiEnumInfo {
    enumtype: Option<String>,
}

#[derive(Debug, Clone, Deserialize)]
struct VppJsApiEnumValueDef {
    name: String,
    value: i64,
}

impl Serialize for VppJsApiEnumValueDef {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(2))?;
        seq.serialize_element(&self.name)?;
        seq.serialize_element(&self.value)?;
        seq.end()
    }
}
``` 
I'm putting all these ine one code block as they are simple struct definition with a custom seriliazer implemented for one of them. 

```rust
[derive(Debug)]
struct VppJsApiEnum {
    name: String,
    values: Vec<VppJsApiEnumValueDef>,
    info: VppJsApiEnumInfo,
}

impl Serialize for VppJsApiEnum {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(1 + self.values.len() + 1))?;
        seq.serialize_element(&self.name)?;
        for e in &self.values {
            seq.serialize_element(e)?;
        }
        seq.serialize_element(&self.info)?;
        seq.end()
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
enum VppJsApiEnumHelper {
    Str(String),
    Val(VppJsApiEnumValueDef),
    Map(VppJsApiEnumInfo),
}

struct VppJsApiEnumVisitor;

impl<'de> Visitor<'de> for VppJsApiEnumVisitor {
    type Value = VppJsApiEnum;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("struct VppJsApiEnum")
    }

    fn visit_seq<V>(self, mut seq: V) -> Result<VppJsApiEnum, V::Error>
    where
        V: SeqAccess<'de>,
    {
        let name: String = if let Some(VppJsApiEnumHelper::Str(s)) = seq.next_element()? {
            s
        } else {
            panic!("Error");
        };
        log::debug!("API message: {}", &name);
        let mut values: Vec<VppJsApiEnumValueDef> = vec![];
        let mut maybe_info: Option<VppJsApiEnumInfo> = None;
        loop {
            let nxt = seq.next_element();
            log::debug!("Next: {:#?}", &nxt);
            match nxt? {
                Some(VppJsApiEnumHelper::Val(f)) => values.push(f),
                Some(VppJsApiEnumHelper::Map(i)) => {
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
        Ok(VppJsApiEnum { name, values, info })
    }
}

impl<'de> Deserialize<'de> for VppJsApiEnum {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_seq(VppJsApiEnumVisitor)
    }
}
```
We are creating a struct `VppJsApiEnum` mostly for parsing enum types in the api definitions fiels, And also implementing custom serde serializer and deserializer with the respective visitor function 
