The VPP binary API is a message passing API. It supports the following API method types.

### Request/Reply
   The client sends a request message and the server replies with a single reply message. The convention is that the reply message is named as `method_name + _reply`. 

### Dump/Detail
   The client sends a "bulk" request message to the server, and the server replies with a set of detail messages. These messages may be of different type. A dump/detail call must be enclosed in a control ping block (Otherwise the client will not know the end of the bulk transmission). The method name must end with method `+ "_dump"`, the reply message should be named `method + "_details"`. The exception here is for the methods that return multiple message types (e.g. `sw_interface_dump`). The Dump/Detail methods are typically used for acquiring bulk information, like the complete FIB table. 

### Events
   The client can register for getting asynchronous notifications from the server. This is useful for getting interface state changes, periodic counters and so on. The method name for requesting notifications is conventionally prefixed with `"want_"`. E.g. `"want_interface_events"`. Which notification types results from an event registration is not defined in the API definition. 

A message from a client must include the `'client_index'`, an opaque cookie identifying the sender, and a `'context'` field to let the client match request with reply. 

### Naming

**Reply/Request** method. Request: name Reply: name+_reply
**Dump/Detail** method. Request: name+_dump Reply: name+_details
**Event** registration: Request: want_+name Reply: want_+name+_reply

### Types

The API supports any C type including compound data types. Conventionally the API is restricted to the basic C types (bool, u8, u16, u32, u64, i8, i16, i32, i64). Currently strings are represented as u8[] as well as an string typedef.

**Type issues**

   - Both binary arrays and strings are represented as u8[]. For example u8 mac_address[6] and u8 build_directory[256]. A separate type for strings has been added.
   - Historically, type checking by language wrappers has been difficult because explicit types weren't used. E.g. an IPv6 address was represented as u8 ip6_prefix[16]. Migration to a dedicated type is underway.
   - Variable length arrays don't include type as in u8 char[0] and the count field is message specific, making it hard to auto-generate code for these message types. The string typedef illustrates the use of a consistent structure.
   - Use of pointers to shared memory e.g. get_node_graph_reply.

**Method issues**

   - Dump/Details do not have an explicit definition of which message types can be returned.
   - Event methods do not have an explicit definition of which messages types can be returned.
   - Dump/Details calls have no way of returning failure to the client.

## Examples 
A request/reply call is structured as follows: 
```C
define show_version {
   u32 client_index;
   u32 context;
};

define show_version_reply {
   u32 context;
   i32 retval;
   string program[limit=32];
   string version[limit=32];
   string build_date[limit=32];
   string build_directory[limit=256];
};
```
With the reply message having the API method name concatenated with the string "_reply". 
A Dump/Details call is structured as follows: 
```C
define map_domain_dump {
  u32 client_index;
  u32 context;
};

define map_domain_details
{
  u32 context;
  u32 domain_index;
  vl_api_ip6_prefix_t ip6_prefix;
  vl_api_ip4_prefix_t ip4_prefix;
  vl_api_ip6_prefix_t ip6_src;
  u8 ea_bits_len;
  u8 psid_offset;
  u8 psid_length;
  u8 flags;
  u16 mtu;
  string tag;
};
```
## Important Links

[VPP/ How to use API trace tools](https://wiki.fd.io/view/VPP/How_To_Use_The_API_Trace_Tools) 

[A fast wireguard mesh using VPP](https://fosdem.org/2021/schedule/event/sdn_vpp_wireguard/) 

[Calico/VPP using VPP with Kubernetes clusters for improved performance and extensibility](https://fosdem.org/2021/schedule/event/sdn_calicovpp/)

[Software Defined Networking by David Mahler](https://www.youtube.com/watch?v=DiChnu_PAzA)






 
