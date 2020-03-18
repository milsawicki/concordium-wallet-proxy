# wallet-proxy

## Credential deployment/account creation

Using the example credential in [examples/cdi.json](examples/cdi.json) the
following displays an interaction that leads to credential deployment.

```console
$ curl -XPUT -d "@examples/cdi.json" localhost:3000/submitCredential
{
  "submissionid": "f1a3a3c17e3bc70a4dad3f409265f8a1fca07c607b732e11cf279dd2e891e0af"
}
```

If the credential is well-formed and communication with the server is good, and
the baker node is alive the server will respond with a submission id which can
be queried later on.

If the submission is successful then the returned `submissionid` can be used to
query the status of the submission as in the following example.

```console
$ curl -XGET localhost:3000/submissionStatus/f1a3a3c17e3bc70a4dad3f409265f8a1fca07c607b732e11cf279dd2e891e0af
{
  {
  "status": "finalized",
  "outcomes": {
    "e9d11ac5b1045092ec5bcbadf5c69b9b4e73f70dc8f3467e37cf2c9b6404e03a": {
      "hash": "f1a3a3c17e3bc70a4dad3f409265f8a1fca07c607b732e11cf279dd2e891e0af",
      "sender": null,
      "cost": 0,
      "result": {
        "accountAddress": "4a3wqcAdVz7QC34rxyXNzRwYMgBKPBNYWfyTBnjAaLpeR6H3pR",
        "outcome": "newaccount"
      },
      "energycost": 9990,
      "type": null,
      "index": 0
      }
    }
  }
```

In this particular case we can see the transaction has already been finalized.

In general the return values possible from the call are
- ```json
  {
    "status": "received"
  }```
  When the submission has been received, but it is not yet committed to any
  blocks and no outcomes are known.
  

- ```json
  {
    "status": "absent"
  }```
  When something went wrong and the submission failed and will not be part of
  any blocks.

- ```json
  {
    "status": "committed",
    "outcomes": { ... }
  }```

- ```json
  {
    "status": "finalized",
    "outcomes": { ... }
  }```

where there can be one or more pairs of block hash and outcome, and outcome
always has the format as above in the case of a finalized block. For
credential submission the format is always going to be the same as above, except that if 
we are deploying a new credential onto an existing account the "outcome" field will be
`newCredential` instead and instead of the `accountAddress` field there will be a `details` field
which contains an object with two fields `onAccount` and `credentialId`, both of which are strings.

If a submission is finalized it will always stay finalized. A received
transaction can transition into either committed, directly to finalized, or to
absent. A transaction that is committed is most likely going to transition to a
finalized one, although it is technically possible that it will become absent as well.

## Simple transfer

When submitting a simple transfer you should make a PUT request to `submitTransfer` endpoint.
The data that should be sent is as the one returned from the library provided as part of the crypto repository.
After submission of the transaction the responses are the same as for the submission of the credential. If successful
a submission id is returned.

When querying the status of this transaction different outcomes are possible. In case of success the following 
will be returned

```json
{
  "status": "finalized",
  "outcomes": {
    "45bee841a3c6f6d5b5aebe6e4193fea7568b2b19e11c9ba8a616074d3fd01610": {
      "hash": "edd9915e22d0c4570a15289912be44b542701d2f245e34d59ea823b2213fae74",
      "sender": "3urFJGp9AaU62fQ3DEfCczqJwVt9V3F1gjE5PPBaYgqBD6rqPB",
      "cost": 10,
      "result": {
        "amount": 1000,
        "to": "3urFJGp9AaU62fQ3DEfCczqJwVt9V3F1gjE5PPBaYgqBD6rqPB",
        "outcome": "transfersuccess"
      },
      "energycost": 10,
      "type": "transfer",
      "index": 0
    }
  }
}
```

The key part is `transfersuccess` as the outcome.

In case of failure for different reasons the `outcome` field can contain one of the following
- "InvalidTargetAccount"
- "NonExistentAmount"
- "MalformedTransaction"
- "InsufficientEnergy"

## Errors

In case of a non 200 return code the return value will always be a JSON object with two fields
```json
{
  "errorMessage": "free form string",
  "error": Int
}
```

Where the error codes currently returned are
- 0 for internal server error, most likely the server could not communicate with the the baker
- 1 for when the request is invalid. There can be a number of reasons for this, but most of them
  should not occur once the initial debugging is over. They indicate that data is malformed in one
  way or another.

## Get account nonce best guess

When making a transfer the wallet must select a nonce for the transaction. This nonce must be sequential.
The server provides a "best effort" functionality for querying the nonce. This is on the endpoint `accNonce`
which will always return a JSON object (unless there is an internal server error) consisting of two fields
```json
{
  "nonce": UInt,
  "allFinal": Bool
}
```

The `nonce` is always the next nonce that should be used provided all the known transactions will be finalized eventually.
- If `allFinal` is `True` then all transactions from this account are finlized and the nonce should be considered reliable.
- Otherwise there are some pending transactions so the nonce returned is a best guess assuming all transctions will be successful.

In case the wallet is the only user of the account then this nonce tracking is reliable. 
If there are other concurrent users of the account then there is a chance the
nonce returned will be wrong, but then it is up to the user to keep track of that themselves.
