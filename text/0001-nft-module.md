## Summary
Spec for a CENNZnet NFT module. It will support the following use cases:  
1) Allow users to create, issue, and manage NFTs using only API tooling on CENNZnet (simple NFT)
2) Allow sophisticated users to customise NFT functionality with smart contracts (smart NFT)
3) Provide a pathway to transition from a simple to smart NFT  

## Motivation
NFTs are a common use case for blockchains. They should be supported by CENNZnet and it's possible to provide a better experience than existing systems by providing this at the protocol level.

## Module Design
To support NFTs without contracts, the module will store NFTs by a class system along with an associated schema definition.  
Applications wanting to interact with a specific NFT may query the onchain schema and will be able to decode the tokens.  
Users must send a tx to create a class and register it's schema and will become the class owner.  
The class owner is able to mint NFTs freely to specific accounts, these will given an incrementing integer ID.  
A combination of (class ID, token ID) will uniquely identify any NFT on CENNZnet.  

## Module API
## Calls
### `create_class(origin, class_id: ClassId, schema: NFTSchema, meta: NFTClassMeta) -> ClassId`
`origin` will become the class owner
`class_id` must be unused
`schema` must have a valid schema
requires a deposit to treasury in order to create the class

### `mint(origin, class_id: ClassId, data: Vec<NFTField>) -> TokenId`
`origin` must be the class owner
`TokenId` incremented

### `transfer(origin, class_id: ClassId, token_id: TokenId, receiver: T::AccountId)`
Transfer NFT ownership to the receiver, without payment
`origin` must be the token holder

### `burn(origin, class_id: ClassId, token_id: TokenId)`
`origin` must be the token holder

### `set_owner(origin, class_id: ClassId, new_owner: T::AccountId)`
Transfer NFT class ownership to a new owner address
Primarily for transferring ownership to a new smart contract or removing ownership i.e set to 0 address

### `buy(origin, class_id: ClassId, token_id: TokenId)`
Buy the specified NFT
- nft must be listed for sale
- origin must be the listed buyer
- funds deducted automatically
emit event `SaleComplete(seller, buyer, class_id, token_id, asset_id, amount)`
or if buyer does not fulfil in time
emit event `SaleIncomplete(seller, buyer, class_id, token_id, asset_id, amount)`
deduct % of sale for class owner as per `secondary_sales_commission`

### `sell(origin, class_id: ClassId, token_id: TokenId, buyer: Address, asset_id: AssetId, price: Balance)`
Facilitates a simple sale to another address for a fixed price
- Module holds the NFT in escrow until funds received
emit event `SaleListed(seller, buyer, class_id, token_id, asset_id, amount)`

### `cancel_sale(origin, class_id: ClassId, token_id: TokenId)`
Cancel the sale listing of an asset

(Extensions for auctions)
### `auction(origin, class_id: ClassId, token_id: TokenId, asset_id: AssetId, reserve: Balance, end: BlockNumber)`
Facilitates an open auction
- asset_id: the asset to accept for payment
- end: blocknumber to terminate
- reserve >= 0
(schedule in future start using schedule module)
- lock highest bidder funds
- release previous higher bidder funds

At auction end, highest bid > reserve wins
- Funds transferred
- NFT transferred
- losing bid funds are unlocked

### `bid(origin, class_id: ClassId, token_id: TokenId, price: Balance)`
- place a bid for an auctioned token
- must lock up origin funds

### `cancel_auction(origin, class_id: ClassId, token_id: TokenId, reserve: Balance, start: BlockNumber, end: BlockNumber)`
- Cancel the auction of the NFT

## Types
```rust
/// Uniquely identifies an NFT class
type ClassId = [u8; 32];
/// An NFTField field type ID
type NFTFieldId = u8;
/// Short description about the field
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

### `Schema(class_id: ClassId) -> NFTSchema` 
Returns the class schema definition

### `ClassOwner(class_id: ClassId) -> T::AccountId`
Returns the class owner

### `TokenOwner(class_id: ClassId, token_id: TokenId) -> T::AccountId`
Returns the token holder

### `AccountTokensByClass(address: T::AccountId, class_id: ClassId, Vec<TokenId>)`
Returns all the token IDs held by a given an address and NFT class

### `Tokens(class_id: ClassId, token_id: TokenId) -> Vec<u8>`
Store the actual token in encoded form. (since schema is dynamic, decoding is a dynamic exercise also)

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

# Smart NFTs
Once an NFTs ownership is given to a contract address it is considered a smart or 'managed' NFT.  
These contracts will define their own public APIs. 
Assuming CENNZnet provides basic NFT creation and management functions, some extended behaviour could be:  

- Create NFT using random seed
Caller calls function like e.g. "hatch" a new NFT is generated using some random data at the current block

- NFT auctions
Start at block X, finish at block Y, highest bidder gets NFT, assuming reserve is met.

## Smart NFT Contract Interface
Contracts deployed to manage NFTs will need to satisfy a standard interface e.g like ERC-721
This will allow the runtime to defer any NFT calls / events to the contract allowing the standard behaviour to be augmented or prevented.
The contract can always opt to keep the standard behaviour.

For example the `transfer` API on all NFTs should still be callable by an api instance using `api.tx.nft.transfer`.
The NFT module would need to add necessary hooks e.g:
```rust
fn transfer(origin, class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> DispatchResult {
    T::ContractEventManager::notify(T::ClassOwner(class_id), RuntimeEvent::NFTTransferStart, class_id, token_id, new_owner)?;

    // module transfer logic...

    T::ContractEventManager::notify(T::ClassOwner(class_id), RuntimeEvent::NFTTransferEnd, class_id, token_id, new_owner)?;
}
```

The smart contract must implement the receiving hooks.
```rust
//! example smart contract code

/// Dispatch a runtime event 
fn on_event(event: RuntimeEvent) -> Result {
    match event {
        RuntimeEvent::NFTTransferStart(a,b,c) => on_nft_transfer_start(a,b,c),
        RuntimeEvent::NFTTransferEnd(a,b,c) => on_nft_transfer_end(a,b,c),
        _ => Ok(()),
    }
}

/// Handle an NFT transfer start event
fn on_nft_transfer_start(class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> Result {
    // check `new_owner` is on an approved list
    // deduct manage fees etc.
    Ok(())
}

/// Handle an NFT transfer end event
fn on_nft_transfer_end(class_id: ClassId, token_id: TokenId, new_owner: AccountId) -> Result {
    Ok(())
}
```
