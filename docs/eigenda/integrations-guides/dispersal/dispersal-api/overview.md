---
sidebar_position: 1
---

# Overview

The EigenDA Disperser is a permissioned, untrusted service hosted by Eigen Labs which serves as a coordinator between clients, the operator set, and onchain Ethereum contracts for the purpose of dispersing __and retriving__ blobs. In the future the EigenDA disperser will be made permissionless, so that clients can interact with the EigenDA network and contracts directly to avoid a liveness dependency on the Eigen Labs disperser.

The source of truth for the Disperser API spec is [disperser.proto](https://github.com/Layr-Labs/eigenda/blob/8ec570b8c2b266fad20ea0af14f0f5d84906c39c/api/proto/disperser/disperser.proto), adjusted to the current release. The goal of this document is to explain this spec at a higher level.

<!-- TODO: Update to point to master, not a specific commit -->

Eigen Labs hosts one disperser endpoint for each EigenDA network. These endpoints are documented in respective network pages [indexed here](../../../networks/README.md).

The EigenDA Disperser exposes 4 endpoints:

1. `DisperseBlob()`
2. `DisperseBlobAuthenticated()`
3. `GetBlobStatus()`
4. `RetrieveBlob()`

These endpoints enable the blob lifecycle, from enqueuing blobs for dispersal to waiting for their dispersal finalization and finally to retrieving blobs from the EigenDA network. The following flowchart describes how move blobs through this lifecycle with respect to these endpoints:

<!-- Flow chart pseudocode -->

```mermaid
graph EigenDA Disperser API Usage Flow;
    A[Blob Ready]
    B[Blob Queued for Dispersal]
    C[Blob Dispersal Finalized]
    D[Blob Expired]

    A --> |DisperseBlob()| B
    A --> |DisperseBlobAuthenticated()| B
    B -->|GetBlobStatus() != finalized| B;
    B -->|GetBlobStatus() == finalized| C;
    C -->|RetrieveBlob()| C;
    C -->|15 days elapses since dispersal finalization| D;
```

The Disperser offers an asynchrounous API for dispersing blobs, where clients should poll the GetBlobStatus() endpoint with the dispersal request ID they received from calling one of the two disperse endpoints until the disperser reports the blob as successfully dispersed and finalized.

## Endpoints

### Dispersal Endpoints

There are two dispersal endpoints, `DisperseBlob()` and `DisperseBlobAuthenticated()`, which achive the same thing under different authentication strategies ( counterintuitively, both endpoints are authenticated in mainnet). Both endpoints accept a [DisperseBlobRequest](https://github.com/Layr-Labs/eigenda/blob/4d19b0597d1151a402e328430a0ff5345a831fb6/api/proto/disperser/disperser.proto#L72) message, with the following fields:

:::warning
It's important to note that during dispersal, blobs are automatically padded with zero bytes such that the final length of the blob in bytes is a multiple of 32. One implication of this is that a naive EigenDA client implementation will not retrieve the exact same blob that was dispersed.
:::

:::warning
Every 32 bytes of data chunk is interpreted as an integer in big endian format where the lower address has more significant bits. The integer must stay in the valid range to be interpreted as a field element on the bn254 curve. The valid range is:

```python
0 <= x < 21888242871839275222246405745257275088548364400416034343698204186575808495617
```

...containing slightly less than 254 bits and more than 253 bits. If any one of the 32 bytes chunk is outside the range, the whole request is deemed as invalid, and rejected. For more detail see [Blob Serialization Requirements](../blob-serialization-requirements.md).
:::

<!-- TODO: Link to schema in code -->

| __DisperseBlobRequest__ |
| Field Name | Type | Description |
| `data` | []byte | The data to be dispersed. The blob dispersed must conform to the [Blob Serialization Requirements](../blob-serialization-requirements.md) which ensure that the blob's KZG commitment may be representative of the original data that was sent to the disperser. |
| `custom_quorum_numbers` | []uint32 | The quorums to which the blob will be sent, in addition to the required quorums which are configured on the EigenDA smart contract. If required quorums are included here, an error will be returned. The disperser will ensure that the encoded blobs for each quorum are all processed within the same batch. |
| `account_id` | string | The account ID of the client. This should be a hex-encoded string of the ECSDA public key corresponding to the key used by the client to sign the BlobAuthHeader. |

<!-- TODO: Follow up on whether this should just be an Ethereum address, not an ECDSA public key as mentioned in the docs. -->

| __DisperseBlobReply__ |
| Field Name | Type | Description |
| `result` | BlobStatus | The status of the blob associated with the request_id. This field is returned in case the blob dispersal queuing fails immediately. If the blob was successfully dispersed, this field will be set to `PROCESSING` (`1`). |
| `request_id` | []byte | The request ID generated by the disperser corresponding to the dispersal. Once a request is accepted (although not processed), a unique request ID will be generated. Two different DisperseBlobRequests (determined by the hash of the DisperseBlobRequest) will have different IDs, and the same DisperseBlobRequest sent repeatedly at different times will also have different IDs. The client should use this ID to query the processing status of the request (via the `GetBlobStatus()` API). |

#### Authentication

The mainnet EigenDA Disperser mandates that clients be authenticated before allowing access to blob dispersal endpoints (`DisperseBlob()` and `DisperseBlobAuthenticated()`). There are currently two supported authentication strategies: IP-based authentication and ECDSA keypair authentication.

##### DisperseBlobAuthenticated(): ECDSA Keypair Authentication

Clients that have registered their EigenDA authentication public key in the form of an Ethereum address with Eigen Labs should do dispersals using the `DisperseBlobAuthenticated()` endpoint, where requests are authenticated via an ECDSA signature. This authentication strategy is recommend over IP-based authentication, which is scheduled for deprecation.

###### Keypair Setup

:::warning
For security reasons, it is strongly recommended that clients define a fresh keypair for EigenDA authentication rather than reuse a keypair associated with an active Ethereum address. EigenDA client implementations currently require the private key of the authentication keypair to be passed in plaintext as a configuration parameter. This is acceptable design for a non-financial authentication token, which is by its nature limited in scope, but not for an Ethereum private key, which provides unlimited access.
:::

Keypair generation can be done using `forge`. First install forge if necessary:

```bash
# Install forge if necessary
# forge install
```

<!-- TODO: set forge install command -->

Then generate your keypair:

```bash
$ cast wallet new
Successfully created new keypair.
Address:     0xb6a5eAD2eAB6f7b5709EBf47D31780cE042BC79d
Private key: 0x82313389b01934d7dd489c25e71cb5e48c34296583fd5906947fc6d6a6b9db8b
```

###### Authentication Flow

At a high level the `DisperseBlobAuthenticated()` endpoint proceeds as follows for each blob dispersed:

<!-- TODO: Insert request diagram -->

1. The client sends a `DisperseBlobAuthenticated()` request along with a DisperseBlobRequest message.
2. The Disperser sends back a BlobAuthHeader message containing information for the client to verify and sign.
3. The client verifies the BlobAuthHeader and sends back the signed BlobAuthHeader in an AuthenticationData message.
4. The Disperser verifies the signature and returns a DisperseBlobReply message.

For detail on how to implement this flow, see [ECDSA Keypair Authentication Flow](./ecdsa-keypair-authentication-flow.md).

##### DisperseBlob(): IP-Based Authentication

Clients that have registered with Eigen Labs using their IP address should do dispersals using the `DisperseBlob()` endpoint, where requests are automatically authenticated by the public IP address of the client. Clients using this authentication strategy should take care to assign their calling service a static IP to ensure stability. This authentication technique is scheduled for deprecation; using keypair authentication instead is recommended.

##### Client Registration

<!-- Maybe move this section to a general Getting Started guide? -->

On testnet dispersal is permissionless for [free tier levels of dispersal throughput](../../../networks/holesky.md), meaning the EigenDA disperser does not require authentication. Clients looking for more throughput should submit a response to the [Testnet Client Registration Form](https://placeholder.vc).

<!-- TODO: Link testnet client registration form -->

On mainnet the EigenDA disperser requires authentication for all throughput tiers, including the free tier. Mainnet clients should submit a response to the [EigenDA Client Onboarding Form](https://placeholder.vc).

<!-- TODO: Link mainnet client registration form -->

### GetBlobStatus()

This endpoint returns the dispersal status and metadata associated with a given blob request ID, and is meant to be polled until the blob is reported as finalized and a DA certificate is returned.

| __BlobStatusRequest__ |
| Field Name | Type | Description |
| `request_id` | []byte | The ID of the blob that is being queried for its status. |

| __BlobStatusReply__ |
| Field Name | Type | Description |
| `status` | [BlobStatus](https://github.com/Layr-Labs/eigenda/blob/master/api/proto/disperser/disperser.proto/#L141) | The dispersal status of the blob |
| `info` | BlobInfo | The blob info needed for clients to confirm the blob against the EigenDA contracts |

<!-- TODO: update URL -->

Since the BlobInfo type has many nested sub-structs, it's easier to describe its schema by annotating an example:

```json

```

<!-- TODO: generate example -->

### RetrieveBlob()

The `RetrieveBlob()` endpoint enables clients to retrieve an individual blob using a `(batch_header_hash, blob_index)` pair, originally derived from inside a `BlobInfo` object returned by `GetBlobStatus()`. Retrieving blobs via the RetrieveEndpoint on the disperser is more efficient than directly retrieving from the DA Nodes (see detail about this approach in api/proto/retriever/retriever.proto). The blob should have been initially dispersed via this Disperser service for this API to work.

<!-- TODO: link -->