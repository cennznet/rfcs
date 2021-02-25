### Goals
- Allow users to create, issue, and manage NFTs using only API tooling on CENNZnet (simple NFT)
- Allow sophisticated users to customise core NFT functionality with smart contracts (managed NFT)
- Provide a pathway to transition from simple to managed NFT

## Design
The CENNZnet runtime should provide a basic NFT module, it may be used *as is* by public calls to create a new NFT, i.e like generic assets and to manage that NFT.  

It should be possible for the NFT module to be called by a smart contract so that NFT functionality can be extended by DApps.  

## Public API
## Calls
### `create_class(origin, class_id: ClassId, schema: Vec<NFTField>) -> ClassId`
`origin` will become the class owner
`class_id` must be unused
requires a deposit to treasury in order to create the class

### `mint(origin, class_id: ClassId, data: Vec<NFTField>) -> TokenId`
`origin` must be the class owner
`TokenId` incremented

### `transfer(origin, class_id: ClassId, token_id: TokenId, receiver: T::AccountId)`
`origin` must be the token holder

### `burn(origin, class_id: ClassId, token_id: TokenId)`
`origin` must be the token holder

### `set_owner(origin, class_id: ClassId, new_owner: T::AccountId)`
Transfer NFT class ownership to a new owner address
Primarily for transferring ownership to a new smart contract or removing ownership i.e set to 0 address

## Types
```rust
/// Uniquely identifies an NFT class
type ClassId = [u8; 32];
/// An NFTField field type ID
type NFTFieldId = u8;
/// Some description about the field
type NTFieldTag = [u8; 32];
/// Ordered fields describing an NFT class)
type NFTSchema = Vec<(NFTFieldId, NTFieldTag)>;
/// Carries NFT field type and data
enum NFTField {
    Bytes32([u8; 32]),
    U128(u128) // default 0
    U64(u64),  // default 0
    U32(u32),  // default 0
    U16(u16),  // default 0
    U8(u8),    // default 0
    I32(i32),  // default 0
}
```

## Storage/Queryable

### `ClassMeta(class_id: ClassId) -> (NFTSchema, Owner)` 
Returns the NFT schema and class owner  

### `TokenOwner(class_id: ClassId, token_id: TokenId) -> T::AccountId`
Returns the token holder

### `AccountTokensByClass(address: T::AccountId, class_id: ClassId, Vec<TokenId>)`
Returns all the token IDs held by a given an address and NFT class

### `Tokens(class_id: ClassId, token_id: TokenId) -> Vec<u8>`
Store the actual token in encoded form. (since schema is dynamic, decoding is a dynamic excercise also)

### `NextTokenId(class_id: ClassId) -> TokenId`
Returns the next available token ID for the given class

### `TokenIssuance(class_id: ClassId) -> u64`
Returns the current number of NFTs in existence for a given class

## Validating Class IDs
Class IDs are 32 byte strings. This is also the ubiquitous Account ID / Hash Type.  
They are chosen over integer IDs for human readability.  
This regex must validate the string
```regex
VALID_CHAR_SET=/^[a-zA-Z\-0-9]+$/
```
The system will accept shorter IDs and pad to 32 bytes with null bytes.  
The prefix `system-` shall be reserved
An empty class ID is invalid i.e. (32 0 bytes) as it is the default empty value.

## NFT Extensions
Assuming CENNZnet provides basic NFT creation and management functions, some example extensions could be:

- Create NFT using random seed
Caller calls function like e.g. "hatch" a new NFT is generated using some random data at the current block

- NFT auctions
Start at block X, finish at block Y, highest bidder gets NFT, assuming reserve is met.

## NFT contract integration
The NFT module will provide basic API calls for contracts to manage NFTs.
The contract must be made aware of events happening to managed NFTs so it can augment or control the behaviour where necessary.

e.g. in the NFT module transfer function something like the following should occur on transfer to ensure the contract has the ability manage the NFTs it owns.

```rust
fn transfer(origin, class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> DispatchResult {
    T::ContractEventManager::notify(T::ClassOwner(class_id), RuntimeEvent::NFTTransferStart, class_id, token_id, new_owner)?;

    // module transfer logic...

    T::ContractEventManager::notify(T::ClassOwner(class_id), RuntimeEvent::NFTTransferEnd, class_id, token_id, new_owner)?;
}
```

The smart contract must implement appropriate receiving hooks
```rust
/// Dispatch a runtime event 
fn on_event(event: RuntimeEvent) -> Result {
    match event {
        RuntimeEvent::NFTTransferStart(a,b,c) => on_nft_tansfer_start(a,b,c),
        RuntimeEvent::NFTTransferEnd(a,b,c) => on_nft_tansfer_end(a,b,c),
        // etc...
    }
}

/// Handle an NFT transfer start event
fn on_nft_transfer_start(class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> Result {
   // ...
}

/// Handle an NFT transfer end event
fn on_nft_transfer_end(class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> Result {
   // ...
}
```
