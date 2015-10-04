- Feature Name: MPID Messaging System
- Type: New Feature
- Related Components: [safe_vault](https://github.com/maidsafe/safe_vault), [safe_client](https://github.com/maidsafe/safe_client), [routing](https://github.com/maidsafe/routing)
- Start Date: 22-09-2015
- RFC PR:
- Issue number:

# Summary

This RFC outlines the system components and design for general communications infrastructure and security on the SAFE Network.

# Motivation

## Rationale

A messaging system on the SAFE Network will obviously be useful.  Over and above this, the proposed messaging system is secure and private without the possibility of snooping or tracking.

A large abuse of modern day digital communications is the ability of spammers and less than ethical marketing companies to flood the Internet with levels of unsolicited email to a level of circa 90%.  This is a waste of bandwidth and a significant nuisance.

## Supported Use-Cases

The system will support secure, private, bi-directional communications between pairs of Clients.

## Expected Outcome

The provision of a secure messaging system which will eradicate unwanted and intrusive communications.

# Detailed design

## Overview

A fundamental principle is that the cost of messaging is with the sender, primarily to deter unsolicited messages.  To achieve this, the sender will maintain messages in a network [outbox][3] until they are retrieved by the recipient.  The recipient will have a network [inbox][4] comprising a list of metadata relating to messages which are trying to be delivered to it.

If the message is unwanted, the recipient simply does not retrieve the message.  The sender will thus quickly fill their own outbox with undelivered mail and be forced to clean this up themselves before being able to send further messages.

This paradigm shift will mean that the obligation to unsubscribe from mailing lists, etc. is now with the owner of these lists.  If people are not picking up messages, it is because they do not want them.  So the sender has to do a better job.  It is assumed this shift of responsibilities will lead to a better-managed bandwidth solution and considerably less stress on the network and the users of the network.

## Implementation Details

The two relevant structs (other than the MPID itself which is a [standard Client key][0]) are the [`MpidHeader`][1] and the [`MpidMessage`][2].

Broadly speaking, the `MpidHeader` contains metadata and the `MpidMessage` contains a signed header and message.  We also want to define some upper limits which will be described later.

### Consts

```rust
pub const MPID_MESSAGE: u64 = 51000;
pub const MAX_HEADER_METADATA_SIZE: usize = 128;  // bytes
pub const MAX_BODY_SIZE: usize =
    ::routing::structured_data::MAX_STRUCTURED_DATA_SIZE_IN_BYTES - 512 - MAX_HEADER_METADATA_SIZE;
pub const MAX_INBOX_SIZE: usize = 1 << 27;  // bytes, i.e. 128 MiB
pub const MAX_OUTBOX_SIZE: usize = 1 << 27;  // bytes, i.e. 128 MiB
```

### `MpidHeader`

```rust
pub struct MpidHeader {
    sender_name: ::NameType,
    guid: [u8; 16],
    metadata: Vec<u8>,
    signature: ::sodiumoxide::crypto::sign::Signature,
}
```

The `sender` field is hopefully self-explanatory.  The `guid` allows the message to be message to be uniquely identified, both by receivers and by the manager Vaults which need to hold the messages in a map-like structure.

The `metadata` field allows passing arbitrary user/app data.  It must not exceed `MAX_HEADER_METADATA_SIZE` bytes.

### `MpidMessage`

```rust
  pub struct MpidMessage {
      header: ::MpidHeader,
      recipient: ::NameType,
      body: Vec<u8>,
      recipient_and_body_signature: ::sodiumoxide::crypto::sign::Signature,
  }
```
Each `MpidMessage` instance only targets one recipient.  For multiple recipients, multiple `MpidMessage`s need to be created in the [Outbox][3] (see below).  This is to ensure spammers will run out of limited resources quickly.

## Outbox

This is a simple data structure for now and will be a hash map of serialised and encrypted `MpidMessage`s.  There will be one such map per MPID (owner), held on the MpidManagers, and synchronised by them at churn events.

This can be implemented as a `Vec<MpidMessage>`.

## Inbox

Again this will be one per MPID (owner), held on the MpidManagers, and synchronised by them at churn events.

This can be implemented as a `Vec<(sender_name: ::routing::NameType, sender_public_key: ::sodiumoxide::crypto::sign::PublicKey, signed_header: Vec<u8>)>` or having the headers from the same sender grouped: `Vec<(sender_name: ::routing::NameType, sender_public_key: ::sodiumoxide::crypto::sign::PublicKey, headers: Vec<signed_header: Vec<u8>>)>` (however this may incur a performance slow down when looking up for a particular mpid_header).

## Messaging Format Among Nodes

Messages between Clients and MpidManagers will utilise [`::routing::structured_data::StructuredData`][5], for example:
```rust
let sd = StructuredData {
    type_tag: MPID_MESSAGE,
    identifier: mpid_message_name(mpid_message),
    data: ::utils::encode(mpid_message),
    previous_owner_keys: vec![],
    version: 0,
    current_owner_keys: vec![sender_public_key],
    previous_owner_signatures: vec![]
}
```

When handling churn, MpidManagers will synchronise their outboxes by sending each included MpidMessage as a single network message.  However, since MpidHeaders are significantly smaller, inboxes will be sent as a message containing a vector of all contained headers.

## Message Flow

![Flowchart showing MPID Message flow through the SAFE Network](MPID%20Message%20Flow.png)

## MPID Client

The MPID Client shall provide the following key functionalities :

1. Send Message (Put from sender)
1. Accept Message header (Push from MpidManagers to recipient)
1. Retrieve Full Message (Get from receiver)
1. Query own inbox to get list of all remaining MpidHeaders
1. Query own inbox for a vector of specific MpidHeaders to see whether they remain in the outbox or not.
1. Remove unwanted MpidHeader (Delete from recipient)
1. Query own outbox to get list of all remaining MpidMessages
1. Remove sent Message (Delete from sender)

If the "push" model is used, an MPID Client is expected to have its own routing object (not shared with the MAID Client).  In this way it can directly connect to its own MpidManagers (or the connected ClientManager will register itself as the proxy to the corresponding MpidManagers), allowing them to know its online status and hence they can push message headers to it as and when they arrive.

Such a separate routing object (or the registering procedure) is not required if the "pull" model is employed.  This is where the MPID Client periodically polls its network inbox for new headers.  It may also have the benefit of saving the battery life on mobile devices, as the client app doesn't need to keep MPID Client running all the time.

## Planned Work

1. Vault
    1. outbox
    1. inbox
    1. sending message flow
    1. retrieving message flow
    1. deleting message flow
    1. churn handling and refreshing for account_transfer (Inbox and Outbox)
    1. MPID Client registering (when GetAllHeader request received)

1. Routing
    1. `Authority::MpidManager`
    1. definition of `MPID_MESSAGE_TAG` and `MPID_HEADER_TAG`
    1. definition of `MpidMessage` and `MpidHeader`
    1. support Delete (for StructuredData only)
    1. support push to client

1. Client
    1. Put `MpidMessage`
    1. Get all `MpidHeader`s (pull)
    1. accept all/single `MpidHeader` (push)
    1. Get `MpidMessage`.  This shall also include the work of removing corresponding `MpidHeader`s
    1. Delete `MpidMessage`
    1. Delete `MpidHeader`


# Drawbacks

None identified, other than increased complexity of Vault and Client codebase.

# Alternatives

No other in-house alternatives have been documented as yet.

If we were to _not_ implement some form of secure messaging system, a third party would be likely to implement a similar system using the existing interface to the SAFE Network.  This would be unlikely to be as efficient as the system proposed here, since this will be "baked into" the Vault protocols.

We also have identified a need for some form of secure messaging in order to implement safecoin, so failure to implement this RFC would impact on that too.

# Unresolved Questions


# Future Work

- This RFC doesn't address how Clients get to "know about" each other.  Future work will include details of how Clients can exchange MPIDs in order to build an address book of peers.

- It might be required to provide Vault-to-Vault, Client-to-Vault or Vault-to-Client communications in the future.  Potential use cases for this are:

    1. Vault-to-Client notification of a successful safecoin mining attempt
    1. Client-to-Vault request to take ownership of the Vault
    1. Vault-to-Client notification of low resources on the Vault

    In this case, the existing library infrastructure would probably need significant changes to allow a Vault to act as an MPID Client (e.g. the MPID struct is defined in the SAFE Client library).

- Another point is that (as with MAID accounts), there is no cleanup done by the network of MpidMessages if the user decides to stop using SAFE.

- Also, not exactly within the scope of this RFC, but related to it; MPID packets at the moment have no human-readable name.  It would be more user-friendly to provide this functionality.

# Appendix

### Further Implementation Details

All MPID-related messages will be in the form of a Put, Get, Post or Delete of a `StructuredData`.

Such `StructuredData` will be:

```rust
StructuredData {
    type_tag: MPID_MESSAGE,
    identifier: mpid_message_name(mpid_message),  // or mpid_header_name(signed_header)
    data: ::util::encode(MpidMessageWrapper),
    previous_owner_keys: vec![],
    version: 0,
    current_owner_keys: vec![sender_public_key],
    previous_owner_signatures: vec![],
}
```

where MpidMessageWrapper can be enumerated as:
```rust
#[allow(variant_size_differences)]
enum MpidMessageWrapper {
    /// Send out a mpid_message
    PutMessage(MpidMessage),
    /// Notify with a mpid_header
    PutHeader(MpidHeader),
    /// List of headers to check for continued existence of corresponding messages in Sender's outbox
    OutboxHas(Vec<mpid_header.name()>),
    /// Subset of list from Has request which still exist in Sender's outbox, all existing headers in inbox to respons GetAll
    HasResponse(Vec<MpidHeader>),
}
```


Requests composed by Client:

| Request Type | Usage Scenario | content | Destination Authority |
|:---|:---|:---|:---|
| PUT    | client A send a msg       | MpidMessageWrapper::PutMessage | MpidManager(A) |
| Get    | client B get a header     |        header_name             | MpidManager(B) |
| Get    |      get a msg            |        message_name            | MpidManager(A) |
| Get    | get all headers           |        A or B                  | MpidManager(A or B) |
| Delete | client B remove a header  |        header_name             | MPidManager(B) |
| Delete | client B remove a message |        message_name            | MPidManager(A) |
| POST   | client A checks existence | MpidMessageWrapper::OutboxHas  | MPidManager(A) |

Note: when a Get request is targeting an account, such account is registered as `online`.


Requests composed by Vault:

| Request Type | Usage Scenario | content | From Authority | Destination Authority |
|:---|:---|:---|:---|:---|
| PutResponse  | put failure (Outbox or Inbox Full) | Error[mpid_message]             | MPidManager(A) | Client(A) |
| Put          | messaging notification             | MpidMessageWrapper::PutHeader   | MPidManager(A) | MPidManager(B) |
| PutResponse  | Inbox full                         | Error[mpid_header]              | MPidManager(B) | MPidManager(A) |
| Get          | fetching a message for push        | message_name                    | MPidManager(B) | MPidManager(A) |
| PostResponse | requested msg not existing         | Error[GetMessage[name]]         | MPidManager(A) | MPidManager(B) |
| Post         | replying the message for push      | MpidMessageWrapper::PutMessage  | MPidManager(A) | MPidManager(B) |
| Post         | pushing a message to client        | MpidMessageWrapper::PutMessage  | MPidManager(B) | Client(B) |
| Post         | replying existing headers          | MpidMessageWrapper::HasResponse | MPidManager    | Client    |


MPID Header:

```rust
/// Maximum allowed size for `MpidHeader::metadata`
pub const MAX_HEADER_METADATA_SIZE: usize = 128;  // bytes

const GUID_SIZE: usize = 16;

/// MpidHeader
#[derive(Eq, PartialEq, PartialOrd, Ord, Clone, RustcDecodable, RustcEncodable)]
pub struct MpidHeader {
    sender_name: ::NameType,
    guid: [u8; GUID_SIZE],
    metadata: Vec<u8>,
    signature: ::sodiumoxide::crypto::sign::Signature,
}

impl MpidHeader {
    pub fn new(sender_name: ::NameType,
               metadata: Vec<u8>,
               secret_key: &::sodiumoxide::crypto::sign::SecretKey)
               -> Result<MpidHeader, ::error::RoutingError> {
        use rand::Rng;
        if metadata.len() > MAX_HEADER_METADATA_SIZE {
            return Err(::error::RoutingError::ExceededBounds);
        }
        let mut guid = [0u8; GUID_SIZE];
        ::rand::thread_rng().fill_bytes(&mut guid);

        let encoded = Self::encode(&sender_name, &guid, &metadata);
        Ok(MpidHeader{
            sender_name: sender_name,
            guid: guid,
            metadata: metadata,
            signature: ::sodiumoxide::crypto::sign::sign_detached(&encoded, secret_key),
        })
    }

    pub fn sender_name(&self) -> &::NameType {
        &self.sender_name
    }

    pub fn guid(&self) -> &[u8; GUID_SIZE] {
        &self.guid
    }

    pub fn metadata(&self) -> &Vec<u8> {
        &self.metadata
    }

    pub fn signature(&self) -> &::sodiumoxide::crypto::sign::Signature {
        &self.signature
    }

    pub fn verify(&self, public_key: &::sodiumoxide::crypto::sign::PublicKey) -> bool {
        let encoded = Self::encode(&self.sender_name, &self.guid, &self.metadata);
        ::sodiumoxide::crypto::sign::verify_detached(&self.signature, &encoded, public_key)
    }

    fn encode(sender_name: &::NameType, guid: &[u8; GUID_SIZE], metadata: &Vec<u8>) -> Vec<u8> {
        ::utils::encode(&(sender_name, guid, metadata)).unwrap_or(vec![])
    }
}
```

MPID Message:

```rust
/// Maximum allowed size for `MpidMessage::body`.
pub const MAX_BODY_SIZE: usize = ::structured_data::MAX_STRUCTURED_DATA_SIZE_IN_BYTES - 512 -
                                 ::mpid_header::MAX_HEADER_METADATA_SIZE;

/// MpidMessage
#[derive(Eq, PartialEq, PartialOrd, Ord, Clone, RustcDecodable, RustcEncodable)]
pub struct MpidMessage {
    header: ::MpidHeader,
    recipient: ::NameType,
    body: Vec<u8>,
    recipient_and_body_signature: ::sodiumoxide::crypto::sign::Signature,
}

impl MpidMessage {
    pub fn new(header: ::MpidHeader,
               recipient: ::NameType,
               body: Vec<u8>,
               secret_key: &::sodiumoxide::crypto::sign::SecretKey)
               -> Result<MpidMessage, ::error::RoutingError> {
        if body.len() > MAX_BODY_SIZE {
            return Err(::error::RoutingError::ExceededBounds);
        }

        let recipient_and_body = Self::encode(&recipient, &body);
        Ok(MpidMessage {
            header: header,
            recipient: recipient,
            body: body,
            recipient_and_body_signature:
                ::sodiumoxide::crypto::sign::sign_detached(&recipient_and_body, secret_key),
        })
    }

    pub fn header(&self) -> &::MpidHeader {
        &self.header
    }

    pub fn recipient(&self) -> &::NameType {
        &self.recipient
    }

    pub fn body(&self) -> &Vec<u8> {
        &self.body
    }

    pub fn verify(&self, public_key: &::sodiumoxide::crypto::sign::PublicKey) -> bool {
        let encoded = Self::encode(&self.recipient, &self.body);
        ::sodiumoxide::crypto::sign::verify_detached(&self.recipient_and_body_signature, &encoded,
                                                     public_key) && self.header.verify(public_key)
    }

    fn encode(recipient: &::NameType, body: &Vec<u8>) -> Vec<u8> {
        ::utils::encode(&(recipient, body)).unwrap_or(vec![])
    }
}
```

To send an MPID Message, a client would do something like:

```rust
let mpid_message = MpidMessage::new(my_mpid: Mpid, recipient: ::routing::Authority::Client,
                                    metadata: Vec<u8>, body: Vec<u8>);

```

Account types held by MpidManagers

```rust
struct Outbox {
    pub sender: ::routing::NameType,
    pub mpid_messages: Vec<MpidMessage>,
    pub total_size: u64,
}
struct Inbox {
    pub recipient_name: ::routing::NameType,
    pub recipient_clients: Vec<::routing::Authority::Client>,
    pub headers: Vec<(sender_name: ::routing::NameType,
                      sender_public_key: ::sodiumoxide::crypto::sign::PublicKey,
                      signed_header: Vec<u8>)>,
    pub total_size: u64,
}
```

General functions

```rust
pub fn mpid_header_name(signed_header: &Vec<u8>) -> ::routing::NameType {
    ::crypto::hash::sha512::hash(signed_header)
}
pub fn mpid_message_name(mpid_message: &MpidMessage) -> ::routing::NameType {
    mpid_header_name(mpid_message.signed_header)
}

pub fn sign_mpid_header(mpid_header: &MpidHeader, private_key: PrivateKey) -> Vec<u8> {
    ::sodiumoxide::crypto::sign::sign(encode(mpid_header), private_key)
}
pub fn parse_signed_header(signed_header: &Vec<u8>, public_key: PublicKey) -> Result<MpidHeader, ::routing::error::Error> {
    ::utils::decode::<MpidHeader>(::sodiumoxide::crypto::sign::verify(signed_header, public_key))
}
```



[0]: https://github.com/maidsafe/safe_client/blob/c4dbca0e5ee8122a6a7e9441c4dcb65f9ef96b66/src/client/user_account.rs#L27-L29
[1]: #mpidheader
[2]: #mpidmessage
[3]: #outbox
[4]: #inbox
[5]: https://github.com/maidsafe/routing/blob/7c59efe27148ea062c3bfdabbf3a5c108afc159c/src/structured_data.rs#L22-L34
[6]: https://github.com/maidsafe/routing/blob/7c59efe27148ea062c3bfdabbf3a5c108afc159c/src/data.rs#L24-L33
