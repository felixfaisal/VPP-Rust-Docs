Fifth 100 lines of code 
```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiCounterElement {
    name: String,
    severity: String,
    #[serde(rename = "type")]
    typ: String,
    units: String,
    description: String,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiCounter {
    name: String,
    elements: Vec<VppJsApiCounterElement>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct VppJsApiPath {
    path: String,
    counter: String,
}
``` 
We are creating few more functions for parsing the API definition files, Next we are defining `VppJsApiFile` which connects all the structs and makes sense. 

```rust
struct VppJsApiFile {
    types: Vec<VppJsApiType>,
    messages: Vec<VppJsApiMessage>,
    unions: Vec<VppJsApiType>,
    enums: Vec<VppJsApiEnum>,
    #[serde(default)]
    enumflags: Vec<VppJsApiEnum>,
    services: LinkedHashMap<String, VppJsApiService>,
    options: VppJsApiOptions,
    aliases: LinkedHashMap<String, VppJsApiAlias>,
    vl_api_version: String,
    imports: Vec<String>,
    counters: Vec<VppJsApiCounter>,
    paths: Vec<Vec<VppJsApiPath>>,
}
``` 
Now the purpose for me doing this code review is to understand what the **union** concept means in the binary APIs, So as you can see `unions:Vec<VppJsApiType>`, So this termed to be VppJsApiType, Let's look at what it looks like. 
```rust
struct VppJsApiType {
    type_name: String,
    fields: Vec<VppJsApiMessageFieldDef>,
}
```
Which has type_name and fields, So there is some clarity right now
Let's have a look at associated functions and methods for `VppJsApiFile` 
```rust
impl VppJsApiFile {
    pub fn verify_data(data: &str, jaf: &VppJsApiFile) {
        use serde_json::Value;
        /*
         * Here we verify that we are not dropping anything during the
         * serialization/deserialization. To do that we use the typeless
         * serde:
         *
         * string_data -> json_deserialize -> json_serialize_pretty -> good_json
         *
         * string_data -> VPPJAF_deserialize -> VPPJAF_serialize ->
         *             -> json_deserialize -> json_serialize_pretty -> test_json
         *
         * Then we compare the two values for being identical and panic if they
         * aren't.
         */

        let good_val: Value = serde_json::from_str(data).unwrap();
        let good_json = serde_json::to_string_pretty(&good_val).unwrap();

        let jaf_serialized_json = serde_json::to_string_pretty(jaf).unwrap();
        let test_val: Value = serde_json::from_str(&jaf_serialized_json).unwrap();
        let test_json = serde_json::to_string_pretty(&test_val).unwrap();

        if good_json != test_json {
            eprintln!("{}", good_json);
            println!("{}", test_json);
            panic!("Different javascript in internal sanity self-test");
        }
    }

    pub fn from_str(data: &str) -> std::result::Result<VppJsApiFile, serde_json::Error> {
        use serde_json::Value;
        let res = serde_json::from_str::<VppJsApiFile>(&data);
        res
    }
}
``` 
One function is  to verify the produced data and another is to retrive from string 

```rust
fn parse_api_tree(opts: &Opts, root: &str, map: &mut LinkedHashMap<String, VppJsApiFile>) {
    use std::fs;
    if opts.verbose > 2 {
        println!("parse tree: {:?}", root);
    }
    for entry in fs::read_dir(root).unwrap() {
        let entry = entry.unwrap();
        let path = entry.path();
        if opts.verbose > 2 {
            println!("Entry: {:?}", &entry);
        }

        let metadata = fs::metadata(&path).unwrap();
        if metadata.is_file() {
            let res = std::fs::read_to_string(&path);
            if let Ok(data) = res {
                let desc = VppJsApiFile::from_str(&data);
                if let Ok(d) = desc {
                    map.insert(path.to_str().unwrap().to_string(), d);
                } else {
                    eprintln!("Error loading {:?}: {:?}", &path, &desc);
                }
            } else {
                eprintln!("Error reading {:?}: {:?}", &path, &res);
            }
        }
        if metadata.is_dir() && entry.file_name() != "." && entry.file_name() != ".." {
            parse_api_tree(opts, &path.to_str().unwrap(), map);
        }
    }
}
``` 
This to parse the APi tree function, I will look into it after I carefully analyze how union looks like in API json files. 
```rust

fn get_rust_type_from_ctype(
    opts: &Opts,
    enum_containers: &HashMap<String, String>,
    ctype: &str,
) -> String {
    use convert_case::{Case, Casing};

    let rtype = {
        let rtype: String = if ctype.starts_with("vl_api_") {
            let ctype_trimmed = ctype.trim_left_matches("vl_api_").trim_right_matches("_t");
            ctype_trimmed.to_case(Case::UpperCamel)
        } else {
            format!("{}", ctype)
        };
        /* if the candidate Rust type is an enum, we need to create
        a parametrized type such that we knew which size to
        deal with at serialization/deserialization time */

        if let Some(container) = enum_containers.get(&rtype) {
            format!("SizedEnum<{}, {}>", rtype, container)
        } else {
            rtype
        }
    };
    rtype
}
``` 
I'm assuming this is to convert c language types to rust types based on the name of the function. 

```rust
fn get_rust_field_name(opts: &Opts, name: &str) -> String {
    if name == "type" || name == "match" {
        format!("r#{}", name)
    } else {
        format!("{}", name)
    }
}
``` 
Needs more analyzing 

```rust
fn get_rust_field_type(
    opts: &Opts,
    enum_containers: &HashMap<String, String>,
    fld: &VppJsApiMessageFieldDef,
    is_last: bool,
) -> String {
    use crate::VppJsApiFieldSize::*;
    let rtype = get_rust_type_from_ctype(opts, enum_containers, &fld.ctype);
    let full_rtype = if let Some(size) = &fld.maybe_size {
        match size {
            Variable(max_var) => {
                if fld.ctype == "string" {
                    format!("VariableSizeString")
                } else {
                    format!("VariableSizeArray<{}>", rtype)
                }
            }
            Fixed(maxsz) => {
                if fld.ctype == "string" {
                    format!("FixedSizeString<typenum::U{}>", maxsz)
                } else {
                    format!("FixedSizeArray<{}, typenum::U{}>", rtype, maxsz)
                }
            }
        }
    } else {
        format!("{}", rtype)
    };
    if fld.maybe_options.is_none() {
        format!("{}", full_rtype)
    } else {
        format!("{} /* {:?} {} */", full_rtype, fld, is_last)
    }
}
``` 
Also needs more analyzing 

```rust
fn camelize(opts: &Opts, ident: &str) -> String {
    use convert_case::{Case, Casing};
    ident.to_case(Case::UpperCamel)
}
``` 
This function probably converts all the cases to be camel cases, As Rust likes to use camel case. 

```rust
#[derive(Clone, Default, Debug)]
struct GeneratedType {
    derives: LinkedHashMap<String, ()>,
    file: String,
    text: String,
}

impl GeneratedType {
    fn add_derives(self: &mut Self, derives: Vec<&str>) {
        for d in derives {
            self.derives.insert(d.to_string(), ());
        }
    }

    fn push_str(self: &mut Self, data: &str) {
        self.text.push_str(data);
    }
}
``` 
This also needs more analysis 

