# Peppermint batch engine

We made this thing for the purpose of minting and transferring NFTs, transferring tez, and potentially other purposes (such as transferring fungible tokens, etc.)

It polls a database for work to do at a configured minimum interval (but will always wait for a batch to be confirmed before pulling another one). Currently it runs with an unencrypted private key in an in-memory signer, it's not nice but it's fast, and the signer is relatively simple to replace if needed (but it has to be able to sign without user input - so hardware wallets won't work, sorry).

## Prerequisites
- node.js version 16+
- npm 7+
- postgresql (tested with version 12)
- [public-keys repo](https://github.com/tzConnectBerlin/public-keys/blob/main/setup-ubuntu.md)

## Installation

In this directory 

`npm install`

create your database schema:

`psql $DATABASE_NAME` < database/schema.sql

## Configuration

`config.json` has all user-configurable parts

## Work queue

The database table used for queuing work is defined as a Postgres schema in `database/schema.sql`.

To add a new work item, fill in the following fields:
- `originator`: The address the operation should be originated from. A process will only pull work with an `originator` value that matches the address of its signer
- `command`: A json structure, with the following fields:
  - `handler`: The name of the module that implements this type of operation 
	  - `nft`
	  - `tez`
  - `name`: The name of the function on the handler that can generate the operation
	  - `create`
	  - `mint`
	  - `create_and_mint`
	  - `create_and_mint_multiple`
	  - `transfer`
  - `args`: The arguments expected by the handler function
	  - `token_id` (integer token id)
	  - `metadata_ipfs` (IPFS URI pointing to TZIP-16 metadata)
	  - `to_address` (Tezos address to which the NFT will be assigned)
	  - `from_address` (Tezos address from which the NFT will be transferred)
	  - `amount` (integer of tokens to transfer)


The parametric statement in pg.js looks like `INSERT INTO peppermint.operations (originator, command) VALUES ($1, $2)`


**Command Functions**

    create: function({ token_id, metadata_ipfs }

    mint: function({ token_id, to_address, amount }

    create_and_mint: function({ token_id, to_address, metadata_ipfs, amount }
    
    create_and_mint_multiple: function({ token_id, metadata_ipfs, destinations }
    
    transfer: function({ token_id, from_address, to_address, amount }

e.g.:

    INSERT INTO peppermint.operations (originator, command) VALUES('tz1L...3g8x', ' {"args": {"amount": 1, "token_id": 2, "to_address": "tz1ar...U9xq", "from_address": "tz1Qu...3WHk", "metadata_ipfs": "ipfs://QmZt...1cEL"}, "name": "create_and_mint", "handler": "nft"} ');


### Command JSON for minting NFTs

```
{
	"handler": "nft",
	"name": "create_and_mint",
	"args": {
		"token_id": 1, // integer token id
		"to_address" : "tz1xxx", // Tezos address to which the NFT will be assigned
		"metadata_ipfs": "ipfs://xxx" // ipfs URI pointing to TZIP-16 metadata
		"amount" : 1 // (optional) integer amount of edition size to be minted
	}
}
```

### Command JSON for transferring NFTs

```
{
	"handler": "nft",
	"name": "transfer",
	"args": {
		"token_id": 1, // integer token id
		"from_address" : "tz2xxx", // Tezos address from which the NFT will be transferred
		"to_address" : "tz1xxx", // Tezos address to which the NFT will be transferred
		"amount" : 1 // (optional) integer amount of tokens to transfer
	}
}
```

### Command JSON for transferring tez

```
{
	"handler": "tez",
	"name": "transfer",
	"args": {
		"amount": 100.0 // Js number tez amount
		"to_address": "tz1xxx" // Address where the tez will be transferred
	}
}
```

In the current state of the codebase, failed operations won't be retried except for a few known retriable fail states (known Octez glitches, no tez in minter account).

**Flowchart of Peppermint.Operations**
```mermaid
flowchart LR
    A[/pending\]-->B[/confirmed\]
    A-->C[/rejected\]
    A-->D[/unknown\]
    

## How to run

`node app.mjs`
