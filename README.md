# MUT Coins on a democratic blockchain
The concept of using voting on a blockchain instead of Proof of Work or
Proof of Stake has yet been impractical, because two questions
could not be solved:

1. Who should vote in a Proof of Voting System?
2. How  to ensure voters do not censor transactions?

In this repository I create a Proof of Concept for how to
create a voting blockchain that can be used for a censorship
resistent coin.

Possible extensions of the Proof of Voting (PoV)
Chains are discussed later.

## PoV Chain
The PoV Chain's blocks have this format:
```protobuf
message Block {
    CID previous = 1; // CID of the previous block. null for genesis.
    CID messages = 2; // CID of the messages object.
    repeated Signature signatures = 3; // signatures of the voters.
    repeated Pubkey new_voters = 4; // public keys of the new
}
```

### Voters
Each voter is a public key pair that must have been added
in a block under `new_voters` within the last `VOTER_BLOCKS`
blocks.

While within the `VOTER_BLOCKS` they can sign any proposed Block,
they have to be added again within those `VOTER_BLOCKS` or they'll
lose that privilege.

### Accepting a new Block
Each Block is proposed and accepted by voters. They collect messages
and propose blocks.

Once a block reaches an absolute majority of current voters, they
are passed.

### Messages
The `messages` CID points to an IPFS object linked to a number
of message objects:

```protobuf
message Message {
    uint32 type = 1;
    bytes payload = 2;
}
```
There can be 2^32 different types and arbitrary data as payload for
messages.

Only the types [0;2^10) are reserved types. Everyting else can be used by
third-party apps. Think of the first 2000 ports in the TCP protocol.

The purpose of the PoV Chain is to provide agreed backward timestamps.

*A backward timestamp is a timestamp that could only have been
created before a certain time.*

## MUT Coins
The first thing implemented using the PoV Blockchain is a payment system
using a digital currency using mutating se crets coins.

MUT Coins are a scheme to create a digital currency without
divulging any information about a transaction to any party not involved
in that transaction except that a transaction exists.

The MUTCoins have these messages:
```protobuf
enum Coins {
    MINT = 0;
    TRANSACTION = 1;
}
```

### Minting a coin (MINT Message)
Any coin is minted by adding a message of type 0 with this payload
to the PoV Blockchain:
```protobuf
message Mint {
    bytes genesisHash = 1;
}
```
The `genesisHash` is the hash of the genesis secret of the coin.
Every coin has a secret, which starts at this genesis secret.
The owner of the minted coin is whoever knows the genesis secret.

### Transacting a coin (TRANSACTION Message)
When a coin is transfered, it's secret and transaction history
is transfered between the previous owner to the new owner.

#### Burning the previous secret
The new owner has to prevent the secret just transfered from being used
in any other transaction, otherwise they'd loose the coin.
Thus they have to derive a new secret from the old secret by hashing
it together with some nonce they randomly generate.

With this new secret, they now have to burn the old secret to prevent
it from being used in any other transaction and without leaking the
actual secret.

This is done by submitting a `Transaction` message with the type = 1
containing the hash of the old secret and hash of the new nonce.
```protobuf
message Transaction {
    bytes secretHash = 1;
    bytes nonceHash = 2;
}
```

#### Verifying the transaction history
Before a received coin can be used, it has to be verified that:
1. The coin was minted by a message on the chain.
2. Each transaction in the transaction history is on the chain.

The transaction history has two parts:
1. The public transaction history, i.e. the list of all `Transaction`
messages accepted to the chain containing the hashes of all secrets
and nonces the coin ever had.
2. The private transaction history. i.e. the list of all the
secrets and nonces that were hashed to create the public transaction
history.

It is the private transaction history that is transfered and kept
alongside the current secret.

In order to verify the private transaction history this algorithm has
to be executed:
```
transaction_history : List<Pair<secret, nonce>>
genesis_secret : secret
// Dict from hash of a secret to a
// list of all transactions with that secretHash
// in the order they appear in the blockchain.
transactions : Dict<secretHash, List<Transaction>>

previous_secret = genesis_secret
for transaction in transaction_history:
    // Is the new secret the result of the old secret
    // and the nonce in the transaction history.
    if transaction[0] != hash(previous_secret + transaction[1]):
        return False

    pub_transactions = transactions[hash(previous_secret)]
    if pub_transactions[0].nonceHash != hash(transaction[1]):
        return False


return True
```

Thi verification ensures, that a secret transfered has been minted on the chain
and that:
1. No coin that was ever double spent during it's transaction history
passes the verification.
2. Leaks from the private transaction history, except for the last secret
and nonce, will not risk the ownership of the coin.

Property one is satisfied because the verification always checks that the
nonceHash of the first transaction for a specific secret is matching
the nonce in the transaction history. If two transactions for the same
secret are submitted, they have to be ordered in the Blockchain and thus
only the first one will pass verification.

Property two is satisfied because once a secret is hashed and put into a
Transaction on the Blockchain, they cannot be used again to transfer coins,
because that'd be double spending - which is prevented by property one.
Thus only the secrecy of the current coin's secret and nonce is important
to protect a coin's ownership.

This does not mean, that the private transaction history of a coin should be
disclosed up until the latest secret, nonce pair - because that'd make it
possible to connect multiple transactions and violate anonymity by
leaking the length of a transaction history.

If Bob knows Alice owned a coin and their private transaction history
was partly leaked, Bob could see, if she still owned the coin thus leaking
information about Alice's account balance.

But because the private transaction history is transfered between owners along
with a coin, there is no gurantee that one of those owners may not leak
eventually. Yet they cannot leak the latest secret and nonce, because those
are always generated by the current owner who has the responsibility to
protect them from being leaked or otherwise someone may submit a transaction
for the coin and thus transfer the coin to themselves.
