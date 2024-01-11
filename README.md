# sOrdinals: Scaling Ordinals on Stacks

> **Disclaimer:** sOrdinals is a protocol in its experimental stage and is currently undergoing discussion and development. 

sOrdinals is a protocol designed to extend the functionality of the Ordinals concept to the Stacks blockchain.
It enables the conversion of Ordinals into sOrdinals via the sBTC bridge,
aligning with the anticipated improvements in transaction speed and cost efficiency following the Nakamoto and sBTC upgrades.

The concept for sOrdinals was initially discussed in this Stacks [forum post](https://forum.stacks.org/t/sbtc-ordinals-sords/14623).
Unlike the SIP-9 standard, which focuses on a singular collection method,
sOrdinals introduces a comprehensive inscription layer to the Stacks network, which allows building additional logic on top and making protocol upgrades.
The SIP-9 wrapper can be added on top for wrapping sOrdinals to open the smart contract use cases as well.
This layer not only facilitates the pegging in of assets from Bitcoin to Stacks but also supports the reverse process – enabling inscriptions made on Stacks to be represented as new inscriptions on the Bitcoin network.

## Key Features

- **sBTC Bridge Integration:** Users can move their Ordinals to sOrdinals and vice versa using the sBTC bridge, enhancing interoperability between Bitcoin and Stacks.
- **High Transaction Speed and Low Fees:** The upcoming Nakamoto and sBTC upgrades promise to make Stacks an even more appealing platform for developers, with faster transactions and reduced costs for bridged assets.
- **SIP-9 Wrapper:** Developing a SIP-9 wrapper for collections on sOrdinals is straightforward and uncomplicated. Additionally, should the bridging process necessitate SIP-9 compatibility, it can be readily achieved with minor enhancements to the protocol.
- **Cost-Effective Inscriptions:** Collection creators can initially inscribe on Stacks and, if required, bridge these inscriptions to Bitcoin. This approach offers significant savings on transaction fees while still ensuring a presence on the Bitcoin network.
- **Expandability and Upgradability of the Protocol:** The protocol is designed to be flexible and open to enhancements suggested by the community. While a SIP-9 wrapper could take the form of a smart contract, it's crucial that the core rules of the protocol remain adaptable and not tied down by any elements that might hinder future upgrades. 
- **Development of a Virtual Machine:** The protocol offers the potential to create a virtual machine. This development could significantly enhance smart contract capabilities within the Stacks ecosystem, allowing for the use of various programming languages. Additionally, it could lead to more gas-efficient smart contract execution.

## How It Works

Like the [STX20](https://github.com/fess-v/stx20-standard) protocol, sOrdinals uses the STX transfer action with a `memo` field for inscription transfers. However, sOrdinals addresses the 34-symbol limitation in memo fields, which is insufficient for most Ordinals use cases except tokens:

- **Unlimited Data Capacity:** sOrdinals achieves increased data capacity by placing the transfer operation in the STX transfer memo field and storing inscription data in smart contract function arguments, allowing to store files more than 1MB in a single transaction (depends on the encoding type). The protocol also enables the storage of data of any size by sequentially merging additional transactions in accordance with its command structure.
- **Batch Operations and Smart Contracts:** This protocol facilitates batch inscriptions and transfers, and is compatible with smart contracts for managing these operations. Significantly, it is not restricted to a specific contract. Instead, it can leverage the capabilities offered by various smart contracts, enhancing its adaptability and utility.
- **Parent-Child Configurations and Delegation:** aligned with Ordinals theory, sOrdinals supports advanced configurations like Parent-Child relationships and delegation.

## Provenance

> **Collection creators are encouraged to use this feature for easy identification of inscriptions as part of a parent collection. The burning of the parent inscription after the minting of the entire collection is recommended to prevent further expansion.**

sOrdinals on Stacks follows a similar provenance approach as the original Ordinals, as detailed in the [Ord repo](https://github.com/ordinals/ord/blob/master/docs/src/inscriptions/provenance.md).

It allows owners to make `child` inscriptions that are linked to a main `parent` inscription. This is useful for organizing collections where each `child` is part of a larger group connected to the `parent`.

## Protocol Specifications

Transfers:
- **Minimal STX Transfers:** Initiating transactions require only a negligible STX amount, even as low as `0.000001`.
- **Normal and Smart Contract calls:** Operations can be performed using both wallet STX transfer and smart contracts.
- **Event Prioritization:** The order of events within a transaction and transactions within a block determines their priority.
- **Format Flexibility:** Inscriptions can support various content encodings and types.
- **Receiver Designation:** The recipient of the STX transfer transaction is set as the receiver for that inscription.
- **Operation Type Indicator:** The first lowercase letter in the memo indicates the operation type.
- **Inscription ID System:** The protocol adopts a 32-byte Inscription ID for all commands, rather than using inscription numbers. This approach is chosen to ensure stability and reliability, particularly in response to the potential reorgs issues.
- **Batch Operations:** Batch inscribes/transfers are possible, but invalid entries invalidate the entire batch. Transfer operations have a lower order priority than the other commands when placed in a single transaction.

Inscribe, bridge, delegate, and other protocol commands:
- **Command Identification:** Each operation is defined by the name of the contract function, which must precisely match the protocol command in a case-sensitive manner.
- **Variable Recognition:** Variables are recognized similarly to commands. They adhere to a strict data structure; any deviation results in the field being disregarded.
- **Command Validation:**  Operations with missing required fields or incorrect data types will be flagged as invalid.
- **Array Consistency:** For operations involving arrays, like `sordinals--recipients` and `sordinals--data-lengths`, the array lengths must be identical to validate the operation. In cases where equal length isn't necessary, such as `sordinals--content-types`, insufficient array size will result in undefined content types for certain inscriptions.
- **Transaction Integrity:** If any command within a transaction is invalid, the entire sequence, including any subsequent transfers, is marked invalid.
- **Batch Operations:** The protocol incorporates specific commands for managing batch processes, including inscriptions, bridge transactions, and delegate actions. These are elaborated in later sections.

**Inscription ID Generation Process**:
The creation of Inscription IDs involves a string concatenation of the transaction hash from the inscription and a sequentially ordered number specific to the sOrdinals protocol events.
The resulting value undergoes a sha256 hash function for finalization.
The order of events follows the sequence set by batch commands, with transfer events being added at the end.
The transfer sequence aligns with the typical order of STX transfer events in transactions.

## Operations

While operations are structured with predefined fields, they're not limited to these. This allows for the inclusion of new fields or types of operations in future upgrades, ensuring flexibility and adaptability.

The protocol operates independently of any specific contract, instead relying on its own set of rules.
This framework allows for the creation of additional utility contracts tailored to specific needs.
These supplementary contracts can be structured to be more appropriate and gas-efficient for particular use cases than in examples, offering enhanced flexibility and optimization.

### Inscribe (only via Smart Contract Calls)

- **Single Inscription:** This feature is designed for inscribing individual files.  Required and optional arguments are described below. 
- **Batch Inscription:** This function supports the inscription of multiple files simultaneously. It combines file data into a singular buffer array, simplifying the process of handling files of different lengths. The `sordinals--data-lengths` field is utilized to segment this combined data back into individual files for indexing purposes. If the data lengths do not match due to insufficient data in the buffer, the entire transaction will be deemed invalid.

##### Clarity example:

```clarity

(define-public (sordinals--inscribe-single
    (sordinals--recipient principal) ;; required
    (sordinals--parent-id (buff 32)) ;; optional, if set incorrectly - tx is marked invalid
    (sordinals--inscribe-events-count uint) ;; optional, if set should be > 0, otherwise tx is invalid
    (sordinals--metadata (buff 1048576)) ;; optional
    (sordinals--encoding (buff 7)) ;; optional
    (sordinals--content-type (buff 100)) ;; optional
    (sordinals--data (buff 1048576)) ;; required
  )
    (ok true)
)

(define-public (sordinals--inscribe-multiple
    (sordinals--recipients (list 1000 principal)) ;; required
    (sordinals--parent-id (buff 32)) ;; optional, if set incorrectly - tx is marked invalid
    (sordinals--metadata (buff 1048576)) ;; optional
    (sordinals--encoding (buff 7)) ;; optional
    (sordinals--content-types (list 1000 (buff 100))) ;; optional
    (sordinals--data-lengths (list 1000 uint)) ;; required, length must be equal to recipients array length
    (sordinals--data (buff 1048576)) ;; required
  )
    (ok true)
)
```


The current version of the indexer supports content encodings like `br`, `gzip`, and `deflate`. More formats will be added in the future. If a file doesn't specify its content type, it will be treated as a binary file for the time being.

Content types, aligned with the Ord indexer's standards, encompass the following array of values:

    `application/json`, `application/pdf`, `application/pgp-signature`, `application/x-javascript`, `application/yaml`, `audio/flac`, `application/ogg`, `video/x-msvideo`, `image/tiff`, `image/bmp`, `video/x-ms-asf`, `video/quicktime`, `audio/mpeg`, `audio/wav`, `font/otf`, `font/ttf`, `font/woff`, `font/woff2`, `image/apng`, `image/avif`, `image/gif`, `image/jpeg`, `image/png`, `image/svg+xml`, `image/webp`, `model/gltf+json`, `model/gltf-binary`, `text/css`, `text/html`, `text/html;charset=utf-8`, `text/javascript`, `text/markdown`, `text/markdown;charset=utf-8`, `text/plain`, `text/plain;charset=utf-8`, `text/x-python`, `video/mp4`, `video/webm`

The fields for content type and encoding are restricted to a maximum of 128 UTF-8 symbols. Should the input exceed this limit, any additional symbols beyond the first 128 will be disregarded.

If a file has a different content type, it will be saved as a binary file.

### Transfer

- **Memo Structure:** `t{inscriptionId}` - inscription ID to be transferred (not the inscription number)

This command can be used from both normal wallets and smart contracts.

Memo construction example using javascript:

```javascript
const hexString = Buffer.from('t').toString('hex') + inscriptionHash; // inscription hash is a 64 symbols long hex identifier

const memo = Buffer.from(hexString, 'hex');

const memoCV = bufferCV(memo);
```

##### Clarity example:

```clarity
(define-public (transfer-single (recipient principal) (memo (buff 34)))
    (stx-transfer-memo? u1 tx-sender recipient memo)
)
```

### Inscribe for larger files

The capability to inscribe larger files is exclusively available in the single inscribe command within the protocol. This is due to practical considerations in the current version, which makes this feature less feasible for batch inscription processes.
The `sordinals--inscribe-events-count` parameter plays a crucial role here, as it determines the number of subsequent events that need to be merged.
To properly execute this function, the value should be set higher than 1.
Setting it to 1 will trigger a standard single inscription event, while setting it to 0 invalidates the inscription transaction.
This setup ensures that larger files are handled effectively while maintaining the integrity of the inscription process.

> **Restriction on Appending Events: The ability to add more events to an inscription is reserved for the wallet that originally performed the inscribing. This holds true regardless of the current ownership status of that inscription.**


When `sordinals--inscribe-events-count` is set to more than 1, subsequent commands can append more bytes to the same file to the data field:

```clarity
(define-public (sordinals--append-inscribe-event
    (sordinals--inscription-id (buff 32)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--data (buff 1048576)) ;; required
  )
    (ok true)
)
```

When enough subsequent events collected, the inscription will be marked as finalized and won't accept secondary events anymore.


### Delegation

> **Deferred Activation Status: The launch of this functionality is currently on hold, primarily due to ongoing discussions about its necessity. The decision regarding its activation, including the specification of an activation block, is still pending.**

Delegation was originally explained in the [Ord repo](https://github.com/ordinals/ord/blob/master/docs/src/inscriptions/delegate.md).

Summarizing, inscriptions can assign a `delegate` inscription. When someone requests the content of an inscription that has a delegate, they will receive the content and type from the delegate instead. This method is an efficient way to produce copies of an inscription.

If users attempt to link an inscription that is already a link, it will directly connect to the original inscription, avoiding the creation of a chain of links.

A similar mechanism will be applied for bridged Ordinals with the exception that the link will be external.

Contract example:
```clarity
(define-public (sordinals--inscribe-delegate
    (sordinals--recipient principal) ;; required
    (sordinals--inscription-id (buff 32)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--metadata (buff 1048576)) ;; optional
  )
    (ok true)
)

(define-public (sordinals--inscribe-deletage-multiple
    (sordinals--recipients (list 1000 principal)) ;; required, length must be equal to inscription-ids array length
    (sordinals--inscription-ids (list 1000 (buff 32))) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--metadata (buff 1048576))
  )
    (ok true)
)
```

## SIP-9 wrapper

Developers have the opportunity to create smart contracts that enable sOrdinals to function as SIP-9 compliant NFTs, thus expanding their applicability.
For such an implementation to be effective and robust, it is anticipated that collaboration among several indexers will be essential to ensure the desired level of durability and performance.

## Recursion

Like the Ord indexer, HTML and SVG inscriptions are sandboxed to avoid linking to external content.
This keeps them unchanged and self-contained.
Yet, there might be exceptions for specific sOrdinals or Ordinals requests.
It is planned to investigate these possibilities in future updates of both the protocol and explorer tools.

## Metadata

Metadata can contain a variety of data types, but the CBOR protocol is preferred due to its human-readable format, which is compatible with explorer tools.

Additionally, metadata can be extended using multi-event inscriptions, effectively allowing for unlimited size.

Further details on the display of metadata in the frontend are available in the [Ord documentation](https://github.com/ordinals/ord/blob/master/docs/src/inscriptions/metadata.md).

## Unsafe content

The developers of the protocol, indexers, and explorers do not have control over the content stored on the blockchain and are not responsible for it.
In the initial version of the explorer tool, unsafe content will be filtered manually.
In future versions, more sophisticated automated filtering methods may be introduced.

## sBTC Bridge Operations

> **Under discussion, very experimental initial idea, which will later be optimized and upgraded.**

The bridging process for sOrdinals and Ordinals involves creating links rather than duplicating data, thus respecting the file size limits of each network.
This approach opens a potential to bridge to other L2s by making only links to inscriptions data to the original network.

More information on the sBTC bridging process can be found [here](https://stacks-network.github.io/sbtc-docs/introduction.html).

The bridging process can also use an alternative approach by utilizing a SIP-9 wrapper directly. This method involves minting a new NFT for each asset being bridged, or releasing an existing one on demand.

### Batch operations

Peg-In and Peg-Out can be done in batches without additional complexity, since both networks support multiple transfers at once.

### Peg-In

Indexer on Stacks verifies that this the operation was sent by the sBTC issuer to validate the bridging operation.

In the bridge operation, the user is required to authenticate and execute the transaction, which includes their Stacks network address specified in the OP_RETURN.
Additionally, the user includes the necessary fees for the bridging process and locks the inscriptions intended for bridging in the issuer's address, thereby initiating the bridge operation.

#### Clarity example for sOrdinals:

```clarity
(define-public (sordinals--bridge-peg-in
    (sordinals--recipient principal) ;; required
    (sordinals--network (buff 100)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--external-inscription-id (buff 128)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--bridge-transaction (buff 4096)) ;; required, if set incorrectly - tx is marked invalid 
  )
    (ok true)
)

(define-public (sordinals--bridge-peg-in-multiple
    (sordinals--recipient principal) ;; required
    (sordinals--network (buff 100)) ;; required, if set incorrectly - tx is marked invalid
    (sordinals--external-inscription-ids (list 100 (buff 128))) ;; required, if set incorrectly - tx is marked invalid
    (sordinals--bridge-transactions (list 100 (buff 4096))) ;; required, if set incorrectly - tx is marked invalid 
  ) 
    (ok true)
)
```

If that ordinal was bridged before, the indexers must unlock the locked inscription,
otherwise a new inscription will be created with a new inscription number assigned. 

### Peg-Out

To begin the bridging process, a user is required to lock their inscription by executing the peg-out function.

```clarity
(define-public (sordinals--bridge-peg-out
    (sordinals--network (buff 100)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--external-recipient-address (buff 128)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--external-inscription-id (buff 128)) ;; required, if set incorrectly - tx is marked invalid 
    (sordinals--bridge-transaction (buff 4096)) ;; required, if set incorrectly - tx is marked invalid 
  )
    (ok true)
)

(define-public (sordinals--bridge-peg-out-multiple
    (sordinals--network (buff 100)) ;; required, if set incorrectly - tx is marked invalid
    (sordinals--external-recipient-addresses (list 100 (buff 128))) ;; required, if set incorrectly - tx is marked invalid
    (sordinals--external-inscription-ids (list 100 (buff 128))) ;; required, if set incorrectly - tx is marked invalid
    (sordinals--bridge-transactions (list 100 (buff 4096))) ;; required, if set incorrectly - tx is marked invalid 
  ) 
    (ok true)
)
```

For the peg-out operation, it's essential that the sBTC issuer initiates the process.
This action must be recognized and tracked by the indexer specifically from the issuer's address. To facilitate user transactions and collect additional fees required for the destination network's transaction,
these preliminary user transactions should be conducted through additional smart contracts provided by the issuer.

For a new Ordinal (first-time bridging from Stacks to Bitcoin) sBTC issuer needs to inscribe:
- JSON file which contains `{ network: 'stacks', linkedInscriptionId: ${linkedInscriptionId} }`, where `linkedInscriptionId` is the original inscriptionId on Stacks.

Peg-Out operations for new Ordinals are expected to be fully compatible with specialized viewing tools.
These tools will be engineered to effectively retrieve and present data from original inscriptions on the Stacks blockchain.
Additionally, they are designed to authenticate whether the issuer corresponds to the original sBTC address.

In order to facilitate this level of verification, it is advisable to develop a dedicated bridge indexer.
This would be akin to the indexers used in BRC20 protocols, incorporating necessary verification logic for assets that have undergone the bridging process.

For an existing Ordinal (previously locked bridged inscription UTXO):
- sBTC issuer unlocks the existing inscription UTXO to the user's wallet.

### Bridge beta

To test the bridging process and the protocol,
a bridge through the [Asigna multisig wallet](https://asigna.io/) will be set up.
Initially, a temporary substitute for the sBTC issuer will be used until the actual sBTC system is operational and ready to handle sOrdinals bridging.
This temporary solution isn't decentralized and won't be as fast as the upcoming sBTC system, but it will provide an opportunity to test the protocol in a real-world scenario and identify the most demanded features, guiding future development.

Upon the establishment of appropriate on-chain indexers, the bridging commands will be updated to incorporate on-chain data validation for the bridging operations. This upgrade will enhance the accuracy and security of the bridging process.

### Support for BRC20 and Advanced Protocols

Protocols such as BRC20, which utilize both an ordinal indexer and a dedicated indexer, can also be integrated for bridging purposes.
However, this integration will necessitate additional efforts, particularly in terms of maintaining and verifying the compatibility and accuracy of the indexers on both the Stacks and Bitcoin platforms.

## Indexer API

Base URL: `https://api.sordinals.com/api/v1`

- Get inscriptions with pagination: `/inscriptions?order={order}&page={page}&limit={limit}`
- Get owned inscriptions with pagination: `/inscriptions/owner/{address}?order={order}&page={page}&limit={limit}`
- Get child inscriptions by parent with pagination: `/inscriptions/parent/{parentInscriptionId}?order={order}&page={page}&limit={limit}`
- Get inscription details: `/inscriptions/{inscriptionNumber}`
- Get inscription details by inscription hash: `/inscriptions/hash/{inscriptionHash}`

Separate domain is currently used to store image content:
- Get content: `https://content.sordinals.com/{internalInscriptionContentId}`, where `internalInscriptionContentId` is obtainable through any of the previously mentioned requests.

> **Caution:** Avoid downloading unknown inscription content as it may contain malicious code.
