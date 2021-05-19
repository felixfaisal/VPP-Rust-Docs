# VPP Binary API Union 

## What is VPP? 
The VPP platform is an extensible framework that provides out-of-the-box production quality switch/router functionality. It is the `open source version of Cisco's Vector Packet Processing (VPP) technology`: a high performance, packet-processing stack that can run on commodity CPUs.

To understand the project better I went through youtube videos of VPP which are <br> 
[VPP Overview]() || [VPP Architecture]() 

## What is Binary API 
VPP provides a binary API scheme to allow a wide variety of client codes to program data-plane tables. As of this writing, there are hundreds of binary APIs.
You can also consider them to be some sort of **psuedocode** which is ingested to produce api bindings for a specific client. Currently there are api bindings for Go, Python, C, C++ 

I went through `.api` files but I feel `.api.json` files are much better as it is a common standard for which there are existing serializers and deserializers, So for further research I only went through `.api.json` files instead of `.api`. 

## What is VPP Binary API Union 
This is a complicated answer especially for me as essentially I couldn't find a detailed description of what it simply meant. If  there was a single line sentence that could help me describe it then i could easily do that but even after reading tons of code I haven't been able to even cook up sentences that could describe what it essentially the concept means, So over here I will describe my process of discovering **VPP Binary API Union** 

Firstly, Going through `api.json` in Rust api generator code [rust-vpp-gen](https://github.com/ayourtch/vpp-api-gen/), I found Unions to be like this 
```javascript
    "unions": [
        [
            "address_union",
            [
                "vl_api_ip4_address_t",
                "ip4"
            ],
            [
                "vl_api_ip6_address_t",
                "ip6"
            ]
        ]
    ],
``` 
So then I went through rust code responsible for parsing and generating code where i found an interesting struct that described `api.json` files in a really good way as to what all exists in it. Wherein I also found `Unions:Vec<`, naturally I wanted to have a look at what the underlying structure is so i looked over it 
```rust
#[derive(Debug, Serialize, Deserialize)]
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
#[derive(Debug, Clone)]
struct VppJsApiType {
    type_name: String,
    fields: Vec<VppJsApiMessageFieldDef>,
}
``` 
The Rust code definetely makes sense but there is no definitive way in which i can explain still what union means. So I went ahead and explored API generator for python which had defined a class for handling Union looks like this, 
```python 
class VPPUnionType(Packer):
    def __init__(self, name, msgdef):
        self.name = name
        self.msgdef = msgdef
        self.size = 0
        self.maxindex = 0
        fields = []
        self.packers = collections.OrderedDict()
        for i, f in enumerate(msgdef):
            if type(f) is dict and 'crc' in f:
                self.crc = f['crc']
                continue
            f_type, f_name = f
            if f_type not in types:
                logger.debug('Unknown union type {}'.format(f_type))
                raise VPPSerializerValueError(
                    'Unknown message type {}'.format(f_type))
            fields.append(f_name)
            size = types[f_type].size
            self.packers[f_name] = types[f_type]
            if size > self.size:
                self.size = size
                self.maxindex = i

        types[name] = self
        self.tuple = collections.namedtuple(name, fields, rename=True)

    # Union of variable length?
    def pack(self, data, kwargs=None):
        if not data:
            return b'\x00' * self.size

        for k, v in data.items():
            logger.debug("Key: {} Value: {}".format(k, v))
            b = self.packers[k].pack(v, kwargs)
            break
        r = bytearray(self.size)
        r[:len(b)] = b
        return r

    def unpack(self, data, offset=0, result=None, ntc=False):
        r = []
        maxsize = 0
        for k, p in self.packers.items():
            x, size = p.unpack(data, offset, ntc=ntc)
            if size > maxsize:
                maxsize = size
            r.append(x)
        return self.tuple._make(r), maxsize

    def __repr__(self):
        return"VPPUnionType(name=%s, msgdef=%r)" % (self.name, self.msgdef)
``` 
Howerver this too didn't make much of sense for me, As I explored I found Go api generator for an older version VPP(19), where they have tackled Union and also produced an example on how to make use of it 

```Golang

```
Example: 

## My Strategy 

- I believe we could do something similar to what Go is currently doing because even though it's a garbage collected it's definitely better than Python in terms of the way code is written 
- There is a need for proper nomenclature to be used and I think GoVPP has done a good job in that regards, We could use or not use the same nomenclature but using it allows new developers to interpret the underlying code better. 
- Since the architecture of vpp-api-gen is still not quite set in stone, I'd like to have a discussion over the use of Rust for generating as parsing through the api.json files has been done by Go and Python, and since what ultimately is required is the output which is the rust files. We don't have to worry about performance as it is a one time process but I'd definitely would like to have a discussion to better understand it. 
- **Mono Repo**, Mono repo would be great to manage all parts of the Rust VPP instead of having different repository for each part. 
