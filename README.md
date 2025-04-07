# Multi-Party Threshold Signature Scheme (TSS) TypeScript impl

A attempt to implement a trustless secure multi-party TSS digital signature scheme for private key management over a network of P2P nodes in typescript, This repo is mainly for learnng purposes to explore and learn about distributed systems and crypto programming. the code in this repo is definitely not meant for production use

## WIP
this work is not complete. so far i have only started working on the keygen step. Signing, key refresh aswell as the p2p and rpc layer have not been implemented yet.

## Scheme Overview
This TSS protocol is based from the paper https://eprint.iacr.org/2021/060.pdf and consists of four phases; one phase for generating the (shared) key which is run once, one phase to refresh the secret key-shares and to generate the auxiliary information required for
signing, one to preprocess signatures before the messages are known, and finally, one for calculating and communicating the signature-shares once the message to be signed is known. my implementation is also based on golang implementation [https://github.com/taurusgroup/multi-party-sig  for guidance.

### Key Generation:
Participants work together to create a shared private key without revealing it entirely. The public key is derived from this collaboration.

### Signing:
To sign a message, participants combine their private key shares in a secure way. The result is a valid signature for the message, verifiable using the public key.

### Key Refresh:
Periodically, the shared private key is updated without changing the public key. This ensures ongoing security, even if some key shares are compromised.

## Whats Done So Far
### Keygen
From following the golang implementation provided paper i have so far nearly implemented the keygen process for secp256k1 keys. On a high level the keygen process involves 5 different rounds where each party computes and validates data that gets shared across the entrie party. the process is initiated by creatimg a paillier key pair from a set of randomly generated
large primes. The Paillier encryption scheme is a probabilistic asymmetric algorithm for public-key cryptography, Its a Homomorphic encryption scheme which allows us to perform mathematical or logical operations on encrypted data. the scheme allows two types of computation:

1) addition of two ciphertexts
2) multiplication of a ciphertext by a plaintext number

Pailliers cryptosystem is used to generate keys because its homomorphic nature ensures better security in later rounds of the keygen process because parties can operate on their encrypted shares using their paillier keys. 

After this each party generates a VSS polynomial of degree T where T is the party length - 1 and with randomly sampled coefficents from a range on the secp256k1 curve. each party can then compute their self secret share by evaluating this polynomial. Each parties share will eventually be used to generate the resulting group key. Lastly each party will then generate a randomId, a schnorr NZIK proof and commit these values. When all of this is done each prarty will broadcast their vss polynomials, schnorr commitment and paillier public key to each other party

Every party will then verifier every other parties message add it to the commits and send this result back before moving onto the next round. Each party now is tasked with generating their own series of zk proofs using their paillier keys. these proofs need to get verified by every other party to ensure the validity of everyones secret share. once all the proofs are verified after some back and forth the final party public key can be constructed from combining and evaluating each partys VSS polynomial. One more proof is generated and verified after this to ensure the party key was constructed correctly

### P2P Network and Http Api
I have also implemented a p2p network and http server to allow nodes to interact over the network. the current state of the p2p network has full peer sync, broadcasting, and direct message capabilities aswell as REST endpoints for individually sending messages to other peers aswell as envoking the TSS protocol. The next step is to handle the keygen generation process over the p2p network.

### In Memory DB with redis
ive implemented an in memory database to store the state of the validators array to Disk such that when a new node joins the network they dont need to provide an configuration about the existing state of the VM before they joined. this in combination with state relplication amoung nodes keeps the validators list up-to-date always amoungst all nodes, new and old.

### practical byzantine fault tolerance consensus
ive started implemented an a distributed blockchain ledger following the practical byzantine fault tolerance method. Right now whenever a keygeneration session has conculded each node will submit a transaction of their final proof to the chain. i have set the current block threshold to be  2f + 1 of the nodes so every time a keygen session concludes a new block will be created purley to server as a public validation of the previous roubds key generation. im not sure whether i will always keep this or not but i will need  ledger later when i start implementing the actual bridge such that users can submit proof of a lock transaction that can serve as ameans to kickstart a signing process.

my current thinking on the topic is to follow an approach similar to Ren protocol whereby we can predertmine sign an event such as a lock deposit and have this signature be able to be reconstricted later from known paramaters such as the transaction payload pHash, the gHash (which from renvm) corresponds to the current gateway shard or collection of darknodes that is currently under management of the keys of said transactions locked funds. whilst this project is small in scope i will probably mock something similar. i might try something similar here. using methods like this the signature signed the VM can be recalcukated and spent in a smart contracts on evm compatabile chains. 

So going forward the next steps for me will be to finish implementing the mpc signing part of the protocol and then start to look int BTC json RPC na perhaos making a wrapper for it such that i can query/initiate relevant transactions. one thing im so far unfamiliar with is how to use bitcoin scripting to my advantage. maybe there is a way to use bitcoin scipting to emit events on such lock depositi addresses.

## Bitcoin API Wrapper
ive started implemening a wrapper for the bitcoin JSON-RPC- spec so that i can watch deposits of a certain address. those addresses being the adress that the kegen algorthm generates. the way bridge will work is each time a deposit to ny of the addressesses get detected we can initiate the signing protoocol. when thats done we can publish this signed transaction/message (where the message is the hash of the btc trandaction itself).to the ledger. after that we just need a set of smart of smart contracts on the ethereum side to co-operareate by recon-structing the transaction and verifyfing that the signare hasnt been spent via a noce and also the recovery address value of the signature should equal the shards address. once this is done i can complete the whoke process. will keep updated


## Usage
to use this and run the keygen process yourself. first clone the repo run 
```
yarn
```
make sure you have redis installed on your machine. if not you can download it from here https://redis.io/download/. Once redis in stalled initialize a local redis instance by running 
```
redis-server
```
then in order to start a new node past the following config into your terminal ```
```
export PORT=portNumber P2P_PORT=p2pPort NODE_ENV=development
nvm use 21.0.0
yarn start:dev
```
You will then have a new node in the P2P network. run this command for maybe 4 or 5 nodes on their own terminal window to set up a demo playground. once done you can start the keygen process by calling
```
POST- http://localhost:3001/start
```
you can start the process from any node but. i have not yet imoplemented a leader election this will come soon. 

### Other methods
```
GET- http://localhost:3001/validators
```
returns a lost of all validators currently active in the network

```
POST- http://localhost:3001/direct-message?id=id
```
send a direct message to a node (test default message will add ability to send custom later)

```
POST- http://localhost:3001/broadcast
```
broadcast a message to all nodes (test default message will add ability to send custom later)

```
GET- http://localhost:3001/get-direct-message?roundId=roundId
```
get all direct messages sent to a particular node for a given roundId. if no value is passed for `roundId`, all direct messages for that node will be returned instead

```

GET- http://localhost:3001/get-message?roundOd=roundId
```
get all broadcast messages sent to a particular node for a given roundId. if no value is passed for `roundId`, all broadcast messages for that node will be returned instead
`

### Demo of keygen so far
https://vimeo.com/manage/videos/903086664/6cec731eb1/privacy?extension_recording=true
