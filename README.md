# FRP (File Registry Protocol) Type 1 Protocol Specification
### Specification version: 0.1
### Date published: November 10th, 2019

## Authors
Jerry Qian

## Acknowledgements
Chris Troutner & Rosco Kalis - for helping me deploy and understand SLP token transactions.

# SECTION I: BACKGROUND

## Introduction

In recent years, people working with blockchain technology have sought for an integration system for referencing and/or storing data on-chain. Ethereum's ENS maps human-readable domain names to blockchain addresses through a 2-layer solution: registry and resolver.  Other emergent systems, such as SKALE Network, use secondary blockchains as integration point for file storage.

We start the approach on Bitcoin Cash on-chain data storage by referencing existing OP_RETURN meta data protocols. The Simple Ledger Protocol (SLP) team is most known for the creation of the Bitcoin Cash-based SLP token specification. Another one of their specifications is BitcoinFiles, which allows for referencing off-chain data storage. 

For parsing OP_RETURN meta data protocols, we will utilize BitDB, a graph-based database for looking up specific transactions and script contents. As all entries into BitDB can be verified against other nodes on the network, it is up to the querier to make the final judgement on valid transaction sets.

Here, we are motivated to present our own solution to on-chain data storage. Our goal is to make the specification extensible for future use cases and implementations.


## Requirements

We believe that a good solution for registering data on-chain should have the following properties:

1. **Permissionless** Its files should not require permission to upload or modify
2. **Simple** The system should be easily understandable and straightforward
3. **Non-invasive** It should require no changes to the underlying Bitcoin Cash protocol
4. **Interoperable** It should work with multiple file storage systems


# SECTION II: PROTOCOL DESCRIPTION

## Protocol Overview

The protocol defines 2 types of on-chain file storage actions, which are contained within the Bitcoin Cash OP_RETURN meta data transactions.

The first  type (UPLOAD) defines and references the data.  The second type (UPDATE) allows users to update the referenced data.

Besides defining a format for the OP_RETURN message, the protocol also defines consensus rules that determine the validity to updating data or data reference.

Note: Currently, Bitcoin Cash OP_RETURN scripts imposes a maximum 220 byte size limit

## Consensus Rules

In all cases of FRP transactions:

* There must be an OP_RETURN output script in the first output (vout=0).

* This OP_RETURN first-output script holds an FRP message, whose contents must conform precisely to this specification.


### Rules for upload transactions (UPLOAD)

* UPLOAD transactions do not rely on the inputs' validity, and are self-evidently valid or invalid.

* UPLOADs require a special `update_vout` output value that associates the data file to an address. This address represents admin ownership and access to the uploaded data.

### Rules for update transactions (UPDATE)

* UPDATE transactions require a special `update_vout`  in the transaction input. Additionally, the transaction output requires the `upload_txid`that the update is referencing. 


## Transaction Detail

### Formatting

FRP uses OP_RETURN script to encode data chunks (byte arrays). Each data chunk inside the OP_RETURN payload is denoted in the following sections using angle brackets **(e.g., &lt;xyz&gt;)**. Messages violating these rules shall be judged entirely invalid under FRP consensus:

1. The script must be valid bitcoin script. Each field must be preceded by a valid Bitcoin script data push opcode. 

2. Each field presented inside the OP_RETURN payload must match the byte size and/or value indicated in parentheses.


### UPLOAD - File Upload Transaction

This is the first transaction which defines the file properties and meta data. The file is thereafter uniquely identified by the file upload transaction hash which is referred to as `upload_txid`.

`update_vout`:  indicates the address allowed to update the file registry


**Transaction inputs**: Any number of inputs or content of inputs, in any order.

**Transaction outputs**:
<table>
<tr>
  <td><b>v<sub>out</sub></b></td>
  <td><b>ScriptPubKey ("Address")</b></td>
  <td><b>BCH<br/>amount</b></td>
</tr>
  <tr>
    <td>0</td>
   <td>
   OP_RETURN<br/>
   &lt;lokad_id: 'FRP\x00'&gt; (4 bytes, ascii)<sup>1</sup><br/>
   &lt;transaction_type: 'UPLOAD'&gt; (6 bytes, ascii)<br/>
   &lt;file_type&gt; (0 to ∞ bytes, suggested utf-8)<br/>
   &lt;file_name&gt; (0 to ∞ bytes, suggested utf-8)<br/>
   &lt;file_hash&gt; (0 to ∞ bytes)<br/>
   &lt;update_vout&gt; (1 byte in range 0x01-0xff)<br/>
   </td>
    <td>any<sup>2</sup></td>
  </tr>

  <tr>
    <td>...</td>
    <td>Any</td>
    <td>any<sup>2</sup></td>
  </tr>

  <tr>
    <td>U</td>
    <td>(U=update_vout) File updater</td>
    <td>any<sup>2</sup></td>
  </tr>

  <tr>
    <td>...</td>
    <td>Any</td>
    <td>any<sup>2</sup></td>
  </tr>
</table>

<sup>1. The Lokad identifier is registered as the number 0x505246 (which, when encoded in the 4-byte little-endian format expected for Lokad IDs, gives the ascii string 'FRP\x00'). Inquiries and additional information about the Lokad system of OP_RETURN protocol identifiers can be found at https://github.com/Lokad/Terab maintained by Joannes Vermorel.</sup>

<sup>2. FRP does not impose any restrictions on BCH output amounts. Typically however the OP_RETURN output would have 0 BCH (as any BCH sent would be burned), and address-related outputs be sent only the minimal 'dust' amount of 0.00000546 BCH.</sup>

### UPDATE - File Update Transaction

Subsequent update transactions are done by spending the `update_vout` UTXO in a special UPDATE transaction. Note that this could be done by someone other than the UPLOAD issuer, if the update authority is assigned to another address.

UPDATE requires the `upload_txid` generated from the initially uploading the file. This transaction id, along with the `update_vout` address, provides the rulesets for who can create valid updates.

**Transaction inputs**: Any number of inputs or content of inputs, in any order, but with required presence of a `update_vout` input (see Consensus Rules).

**Transaction outputs**:
<table>
<tr>
  <td><b>v<sub>out</sub></b></td>
  <td><b>ScriptPubKey ("Address")</b></td>
  <td><b>BCH<br/>amount</b></td>
</tr>
  <tr>
  <td>0</td>
    <td>OP_RETURN<BR>
&lt;lokad_id: 'FRP\x00'&gt; (4 bytes, ascii)<BR>
&lt;transaction_type: 'UPDATE'&gt; (6 bytes, ascii)<BR>
&lt;upload_txid&gt; (32 bytes)<BR>
&lt;file_type&gt; (0 to ∞ bytes, suggested utf-8)<br/>
&lt;file_name&gt; (0 to ∞ bytes, suggested utf-8)<br/>
&lt;file_hash&gt; (0 to ∞ bytes)<br/>
&lt;update_vout&gt; (1 byte in range 0x01-0xff)<br/>
  </td>
    <td>any</td>
  </tr>

  <tr>
    <td>...</td>
    <td>Any</td>
    <td>any</td>
  </tr>

  <tr>
    <td>U</td>
    <td>(U=update_vout) File updater</td>
    <td>any</td>
  </tr>
  <tr>
    <td>...</td>
    <td>Any</td>
    <td>any</td>
  </tr>

</table>


# SECTION III: FURTHER ANALYSIS

### Proxies

Clients can query the proxy BitDB database and get a judgement on any given transaction. Ultimately, however, it is up to the querier to trust the response. Ideally, the response data should be cross-referenced from multiple sources to get a higher confidence degree of a valid transaction.


### Linking Transactions

OP_RETURN meta data protocols are not recognized by network nodes, and thus require extra consensus rules participants must agree upon. The initial file upload provides a transaction id that helps to uniquely identify each file registry on-chain. This transaction id provides a UTXO-like link for validating the genesis point of the file upload.

Whenever there's a new transaction or block, BitDB parses its OP_RETURN into its JSON serialization format as specified in https://bitdb.network. This, in conjunction to FRP rule parameters, creates a state machine that listens to these events, and processes them in accordance to the FRP rules for state transition. The state is updated into the graph database, creating an up-to-date global state. The entire global state is kept up-to-date by the blockchain, and can be recreated at will by new entrants.

Once we have a verifiable global state for the protocol, it becomes easy to build various applications that take advantage of the flexible graph queries. Valid state transitions in the FRP system goes as follows:

1. The state machine maintains a graph data structure with many OP_RETURN transaction chains stemming from the main chain.
2. When an address makes a FRP transaction **(UPLOAD or UPDATE)**, the transaction inputs and outputs are validated against  the FRP format specification.
3. The state machine also checks other FRP-valid outputs from the global state to make sure no "double-spending" is recognized. 


# References
`BitDB`: https://docs.fountainhead.cash/docs/bitdb
`Simple Ledger Protocol`: https://github.com/simpleledger/slp-specifications/blob/master/slp-payment-protocol.md

# Copyright

This protocol specification is published under the terms of the MIT license.
