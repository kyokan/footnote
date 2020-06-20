# Footnote Protocol Ideas

This document outlines several areas of potential improvement. It is not meant to be a recommendation or roadmap.  The community is free to elaborate on the changes proposed here or explore new designs.  The community is also feel free to disregard any of these ideas or explore/expand the [PIPs](https://github.com/kyokan/footnote-PIPs).

## Note on P2P
* It is suboptimal to use protocols such as libp2p (or bittorrent), as it is focused on DHT/pubsub, instead of a broadcast medium. It uses protobuf as a dependency (a lack of canonicality for cryptographic proofs), as well a heavy dependency upon library quirks, making it exceptionally difficult for it to be portable and provable in embedded devices or other languages (in the face of adversaries exploiting library consensus faults). These types of protocols are useful as a parallel layer, e.g. metadata can be stored as a pointer to larger content in DHT-based networks. Additionally, differences in protocol design is necessary for this specific use case (as network protocol design will be the biggest factor in performance). An example would be making the protocol ensure high throughput in a multicast layer 3 with high number of messages per second (satellite broadcasting all traffic to a wide area).

# Subdomains
* A domain actually only maintains a 64KB record of up to 256 (uint8) subdomains. 
* It contains a list of subdomains. Conceptually, the data structure can be: uint32 idCreateAt, uint8 highestSectorPosition, [64]byte subdomainname, [32]byte pubkey.
* The idCreatedAt is a unique id and timestamp combination. If one wants to create 256 records immediately, just submit 256 spaced 1 second apart. The records *MUST* have incrementing idCreatedAt messages (not out of order).
* There are a maximum of 256 sectors to allocate (64KB each). The highestSectorPosition is the number of sectors allocated plus the previous highest. E.g. if the first record is 3 and the second record is at 5, then that means that the first record has 3 sectors of usable space and the second sector has 2 sectors of usable space.
* A fixed allocation per slot (256bytes) and sets the ID to its position would be more difficult when incremental updates are used. If data is pruned/compacted, then the IDs will implicitly change, making fixed IDs much more challenging. It, however, may still be easy to set a fixed size per slot for easier scanning.
* The wire protocol should reflect that it is updating subdomains instead of domains. When a domain gets updated, the subdomain entries get added as well.
* Subdomain records are treated as WRITE ONLY, and can only be reset with a new on-chain Update message on Handshake (with a new TXT record pubkey) or RENEW messages.
* The highest subdomain id value is the one assigned to duplicate subdomain names. Duplicate IDs should not exist, if they exist both should be ignored.
* Subdomain pubkey changes are presumed to be a new record, so there will be a space cost for key rotation. The old record bytes are still reserved from future use and are effectively consumed until there is an on-chain update. There's no clear way to safely "delete"/"move" records, as a equivocation of multiple incremental updates (see below) with a fraud proof can be produced at any time, removing the ability for an overwrite to be correct. The code for this sounds too annoying, so it's better to be kept simple. The result would mean if a subdomain wants to use a new pubkey, for example (e.g. key lost), they have to create a new record with the same name -- the old space used is still consumed.

# Signature Commitments
* The domain owner's signature (and subdomain's) signature should commit to the name. Otherwise, a 3rd party can create a Handshake TXT record which uses the same pubkey. E.g. Imagine if .alice wants to use her name. Then a 3rd-party troll who owns .badmeme points to the same key.

# UPDATE message fix
* Currently, UPDATE and UPDATEREQ may be suboptimal for bandwidth efficiency. It is better to relay only the current timestamp when one has a newer update available. Send a notification of the timestamp available, and the peer can respond with an UPDATEREQ if desired.

# Incremental Updates
* The general idea here is to shift from a random access/update to an append-only log, which can be reset to zero with a new epoch. This may increase performance significantly, as it means that the P2P messages will be much smaller, as well as easier to process (only need to read the updates).
* All updates are in 256-byte increments. Updates which are not exactly divisible by 256bytes will result in a little bit extra space "wasted" (which can be compacted in the future; in the next epoch).
* Merkle proof is not needed for incremental updates. 64KB pages. 256byte sectors. Each sector is hashed into 32bytes, and incrementally hashed. The final sector when 64KB have been filled, becomes the page hash. Each page hash is incrementally hashed forward. This still allows for submitting the entire dataset in 8192B series of hashes (256 hashes representing 64KB each) as currently designed.
* Entire pages can be downloaded similar to the current implementation. However, incremental updates are possible within pages. Simply include the current tip hash, and the peer will respond with the remaining data. This is fully verifiable, as any equivocation should be recognized by the peer (and may be responded with a proof that the tiphash does not match the peer's record).
* New epochs mean that the entire data is reset back to nothing. The incremental log starts over. This is used when the blob fills up.

# Banning and recovery of invalid incremental updates
* A problem arises when a subdomain submits an invalid incremental update. One cannot do a temporary ban, as a malicious subdomain could backdate their timestamp and do many frequent invalid updates. One cannot ascertain for how long one can ban for, as well as whether one is receiving an old replay from a peer.
* Therefore, any subdomain which submits an invalid incremental update should have that ID banned via a proof.
* Include a domain record, domain header for that record, subdomain headers, and proof of invalid incremental update (two subdomain headers).
* Anyone requesting ID will respond with the ban information.

# Header formats

Domain update header: 
1. [32]byte domainHash
  * Hash is: a hash of the top level domain name 
  * Most not be a hash of a TLD which contains periods
2. [32]byte currentTxNormalizedHash
  * This value must equal the current handshake record value of the most recent output of this name on Handshake.
  * After the update record is synced (hours later), this value is stale, then it is treated as stale/invalid and rejected/ignored. we want to make it so that a domain update doesn't overlap with the old one, as this is an append-only log. It is the node's implementation responsibility to expire the old name record when there is an RENEW/UPDATE/etc.
  * We do not use the blockheight since this cannot be presigned. Meaning, the transaction blockheight cannot be known ahead of time. If the name is owned by a multisig, then the multisig group should agree and sign on the domain record before they update the name record. Otherwise, if blockheight was used as the commitment and they end up disagreeing later (or go offline), they will not have any records, resulting in very unhappy subdomain users. This resolves that, by committing to the transaction (normalized, with the witnesses stripped) as one knows the transaction id beforehand. This can be verified on a light client as well. Note that the update messages only need to provide the blockheight, but the SIGNATURE is signing the currentTxNormalizedHash.
  * Sync speed is not immediate as that introduces consensus faults, it may look like it works when one tries to make record updates fast, but it's a trivial DoS vector. As how it works now, the updates happen periodically every n handshake block heights.
  * If there is a RENEW/UPDATE, the TLD owner *SHOULD* send a new header/record within the update period to ensure that the data can continue to be available.
  * If the current state is TRANSFER/FINALIZE/REVOKE or any bidding state, then the record is presumed to be valid until expiration or a new RENEW/UPDATE transaction.
3. uint8 size
  * Size of the record in 256 byte chunks.
  * Each name record consumes 256 bytes exactly.
  * This *MUST* be above any prior size unless the hnsRecordBlockheight is incremented/updated.
  * Update records, therefore, should be querying (domain, hnsRecordBlockeight) and receive a return value of a size.
4. [32]byte hash
  * 256-byte chunk hash commitment, see below for hashing format
5. [64]byte signature
  * The signature commits to: domainHash, timestamp, hash.

Domain Update notification:
1. domainHash
2. uint32 currentTxBlockheight
3. size

If one receives a domain update and (epoch, size) is:
* Is lower than what you have: respond with a domain update notification of your own
* Is equal: do nothing
* Is higher: queue with a download request.

Domain request response wire message (different than the header):
1. domainHash
2. uint32 currentTxBlockheight: This is different as the currentTxHash can be derived (note to account for duplicates). What this provides is a clean notification/indicator when the epoch resets
3. size
4. hash
5. signature
6. list of sector hashes

Subdomain update header:
1. [32]byte domainHash
  * Hash is: a hash of the top level domain name
  * See domain header for more information
  * Does not commit to the RENEW/UPDATE, since we want the record to persist and
    validate without a new record.
  * *MUST NOT* be a hash of a TLD which contains periods
2. uint32 idCreatedAt
  * See Subdomain section above. This is the unique subdomain ID.
  * If this is an unrecognized idCreatedAt, then the node does not process it.
3. uint32 epochTimestamp
  * This subdomain's epoch timestamp (when the blob was last reset).
  * Can be updated by the subdomain owner at will without contacting the domain owner.
  * Nodes can set policies on minimum time before accepting a new epoch (e.g. only allow one new epoch every 2 weeks), should maintain a local copy of when the epoch was last updated locally to keep track of when to accept, that local timestamp should be used *NOT* 2 weeks after the epoch timestamp.
  * This *MUST* be above any prior timestamp
  * This *SHOULD* be equal or above the timestamp in the TLD's subdomain record
  * This *MUST NOT* be more than 3 hours in the future
4. uint16 size
  * Size of the record in 256-byte chunks.
  * Since this is an increasing record (and the timestamp is in messages), there is no need for a timestamp in the header.
  * This *MUST* be above any prior size unless the epochTimestamp is incremented.
  * Update records, therefore, should be querying (domain, idCreatedAt, epochTimestamp) and receive a return value of a size.
5. [32]byte hash
  * The sequentially hashed contents of all current blob data
  * Commits to all 256-byte chunk hashes, see below for hashing format
6. [32]byte messageRoot
  * Merkle root of all messages, size 2^16. If there are over 65536 messages, then the subsequent messages are ignored.
  * This *MUST NOT* be enforced on the protocol level. If someone submits garbage or all zeroes, it is fine. Their message simply cannot be validated on light clients. The primary people being hurt by this is the owner of the subdomain; propogation is only affected by the sectorHash.
  * Even though it is not validated, it must be sent over the wire to build the signature.
7. [64]byte signature
  * The signature commits to: domainHash, idCreatedAt, epochTimestamp, size, sectorHash, messageRoot. (less than 128 bytes per chunk so this is fine)
  * Both the name and ID are committed to since the domain owner could assign the same pubkey to a different name in the future.

Subdomain update notifications occur in a manner similar to domain updates.
Subdomain Update Notification:
1. domainHash
2. idCreatedAt
3. epochTimestamp
4. size

# Fraud Proofs

The fraud proofs contain:
* Domain/Subdomain Header 1
* Height and Sector Hash of the equivocating sector for Header 1
* Subsequent (64KB hashed) sector checkpoints to generate the proofs for Header
  1
* Domain/Subdomain Header 2
* Height and Sector Hash of the equivocating sector for Header 2
* Subsequent (64KB hashed) sector checkpoints to generate the proofs for Header
  2

When a fraud proof is validated, the record is permanently banned until:
* If it is a subdomain, a new valid domain record contains the same name
* If it is a domain, then a new committed currentTxNormalizedHash is used
  (Handshake mined a new transaction for that name)

This record will be relayed when others request this name.


# Sector Hash

Currently, the design uses a merkle tree of 64KB sectors. Instead, it may be easier to use a serial hash within each sector. The sectors themselves are then serially hashed.

Within each 64KB sector, every 256-bytes gets hashed in a 32-byte hash. Those 32-byte hashes are sequentially hashed with the previous hash. Messages below 256-bytes are padded with zeroes to be 256-bytes exactly.

For each sector, there are 256 32byte hashes. These sectors are then serially hashed to produce a final hash.

There can be minimal changes over the wire, as every 64KB is hashed and the standard UpdateResponse message includes all sector hashes. By downloading 256 * 32B = 8192B similar to the UpdateResponse message, one can do chunked updates. One simply takes the previous checkpoint hash, and hashes the 64KB of data, which should produce the next checkpoint.

This structure is ideal for incremental updates.

#### Example pseudocode

This is an example of appending 256bytes at a time (of course, should be optimized to do an arbitrary amount of data).

Input: hash of existing blob, new data being added (256 bytes at a time)
Output: new final hash

```
  addHash([32]byte prevSectorHash,
      [32]byte prevHash,
      [256]byte input,
      uint16 pos):
    [32]byte sectorHash = blake2(append(prevHash, input))
    if ((pos + 1) % 256 != 0)
      return sectorHash, nil
    else
      //The second part is what goes into the 8192byte UpdateResponse
      //The hashes can be recreated using a series of these
      return blake2(append(prevSectorHash, sectorHash)), sectorHash
```

When it fills up a 64KB sector, the first value is the value stored as the tip hash of the blob (all items hashed). The second return value gets checkpointed to disk. For a maximum size of 16MB, a series of the second return value can produce sufficient data to reproduce the current hash. E.g. if one only wants the 3rd sector, they can get 64KB for the 3rd sector, then take in the sectorHash for the 2nd sector and running addHash sequentially should produce the hash for the 4th. By having the 8192B UpdateResponse, any arbitrary sector can be verified. The only added expense is storing these 8192bytes as well as an additional hashing step every 64KB.

Might need to be tweaked for the last sector to be hashed again as well for smaller proof sizes, but increases the lines of code and storage.

# Multisig 
* Multisig is desirable for domain ownership as some may want a quorum of 30 validators to own a name and to update the domain records (list of subdomain owners), or for the domain owners to also be a quorum of subdomain owners.
* Could be done with straight [ECDSA using GG20](https://eprint.iacr.org/2020/540.pdf).  GG20 requires no changes in the protocol to make it multisig.
* Schnorr can be used with fewer cryptographic assumptions, with the penalty of larger proofs and a larger consensus change). The Schnorr pubkey script can be signed by including a subkey as the message, the signature the previous bullet point's key, and the subkey's signature of the message itself. This enables attributable schnorr m-of-n in around 160 bytes. Could include a time range of when a subkey is valid.
* Alternatively, one could consider doing this using plain-jane ECDSA at the application level. E.g., modifying the TXT records on HNS to include the validator set, and checking to see if X of Y normal signatures validate.  This avoids compatibility problems introduced by using newer crypto, since it may be hard to find and integrate well-vetted crypto libraries for Schnorr or GG20 in common languages Footnote developers may want to use.

# Light Client Validation
* The highest right branch committed to could be a commitment of all current messages (each leaf is a message hash)
* There is NO guarantee on the validity of this message. If a user decides to commit to something crazy, they can. They're the ones primarily harmed by this, if they  find some weird use case for it, who cares. It's assumed if someone wishing to provide light client proofs, that they must have all the data to provide it.
* For subdomains, the TLD owner should also commit to a merkle tree of all subdomains.
* Full light client proofs provide the TLD header, subdomain message (in the domains' 64KB blob), merkle proof to subdomain ownership record, subdomain header, merkle proof to message, and the message itself. Much of this data, of course, can be cached client-side as multiple messages from the same subdomain would reuse a lot of this data.

# Notes
* 256 subdomains per domain * ~50MM domains = 12,800,000,000 (12.8 Billion) total records in the worst-case. Each domains can be stored in memory as a 1KB blob of 256 8-byte timestamps. That leaves a record of 50MM in an index. Should be VERY fast. Total memory cache for P2P performance is around ~100GB for high performance storage in RAM/SSD. Timestamps could be cached in high performance implementations since it's assumed the primary message will be notification of peers updates of which names have updated and their timestamp, with a LOT of duplicate messages from many peers.
* There is probably an additional wire message optimization possible for subdomain updates for data retained between epochs, keyed by message timestamp (message timestamps MUST be unique for this to work). A node can always do a full redownload in the event this fails.
* If the UPDATE/RENEW behavior is as described above, someone selling a TLD can update a record with an UPDATE message first to the owner, before doing a TRANSFER.
* Subdomains on the same chain is silly and poorly thought out, as it's no different than a TLD (the dot is to define a different scaling/trust-model/hierarchy). This protocol is about resource constraints, therefore having defined limits is necessary. The limits on the number of TLDs exist in HNS likely due to the cryptoeconomic constraints of maximizing value of names, which maximizes long-term security and value (RENEW transactions function as a lower bound on the lowest value name when the mempool is always full, as there are a limited number of RENEWs per block).
