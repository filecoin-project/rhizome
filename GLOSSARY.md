# Rhizome

Rhizome is a managed service provider (MSP). It improves the durability of Filecoin Onchain Cloud storage by managing where clients' data is stored, how it is replicated and repaired, and the storage relationships under which it is paid for.

## Language

**Client**:
The party obtaining managed storage from an MSP.
_Avoid_: User, when referring to the party to the storage deal

**MSP**:
The service provider responsible to a client for managing the preservation of their data across storage providers.

**Durability**:
The probability that stored data remains recoverable over its intended lifetime despite failures.

**Storage provider (SP)**:
A provider holding data under a storage relationship managed by the MSP.

**Approved provider**:
An SP in the approved provider set used for replica selection.

**Endorsed provider**:
An SP in the endorsed provider set, carrying higher trust and stronger requirements regarding cooperation.

**Upload SP**:
The SP selected to receive a client's initial upload, trusted to hold the sole available copy while storage commitments and replication are established. This is a role an SP takes for an upload, not a separate kind of provider.

**Storage terms**:
The immutable commercial terms governing a storage deal, including its price and payment currency. Client-to-MSP and MSP-to-SP deals can use different storage terms.

**StorageTermId**:
The identifier of a particular set of storage terms.

**Storage deal**:
A storage relationship between two parties governed by selected storage terms. The client-to-MSP deal is distinct from the MSP-to-SP deals used to fulfill it.

**Piece**:
A content-addressed unit of data identified by a PieceCID. Its identity is separate from the storage deals and datasets in which it participates.

**Client interest**:
A client's requirement that MSP maintain protection for a piece under a client-facing storage relationship.

**Placement**:
A specific storage relationship for a piece at an SP, including its backend dataset membership once committed. It is the unit of repair, may serve multiple client storage deals, and remains distinct from other placements of the same piece at the same SP.

**PDP dataset**:
A collection of committed pieces covered by a PDP proving relationship. Client-facing and backend datasets have a many-to-many relationship, with backend datasets pooling pieces from multiple clients.

**Repair**:
Restoration of the required storage protection for managed data. Initial replication and replacement of failed copies are instances of the same process.

**Repair-in-place**:
Restoration of a missing piece at its current SP, preserving the placement rather than replacing it with one at another SP.
