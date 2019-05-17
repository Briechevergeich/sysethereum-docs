# Syscoin <=> Ethereum bridge docs

The Syscoin <=> Ethereum bridge is a system that allows Syss to be moved from the Syscoin blockchain to the Ethereum blockchain and back.


## Main subprojects
* [Sysethereum contracts](https://github.com/syscoin/sysethereum-contracts): Ethereum contracts.
* [Sysethereum agents](https://github.com/syscoin/sysethereum-agents): External agents.
* [Sysethereum Dapp](https://github.com/syscoin/sysethereum-dapp): UI Dapp for reference implementation.
## Sys to Eth

![Design](./sys2eth.png)

## Superblocks

The Sys -> Eth side uses a new concept we named Superblocks. Read the [white paper](superblocks/superblocks-white-paper.pdf).


## Eth to Sys

![Design](./eth2sys.png)

## Bridge

We implemented a "non-collateralized" solution for the Eth -> Sys side. Here are the core concepts:

* When a user wants to get sysx tokens, she has to send the sys to a Superblock contract which relays to a Token contract. The smart contract mints sysx tokens for the user. Assets on Syscoin are also done in similar way.
* When a user burns her sysx tokens, she has to wait for 240 confirmations and then create a mint transaction on Syscoin to request the sys back to the user.

## Actors

This is the list of external actors to the system and what they can do.

* User
  * Lock (Sys -> Eth).
  * Transfer sys tokens (Eth -> Eth).
  * Unlock (Eth -> Sys).
* Superblock Submitter
  * Propose superblocks.
  * Defend superblocks.
* Superblock Challenger
  * Challenge superblocks.


## Workflows
* New Superblock
  * There is a new block on the sys blockchain, then another one, then another one...
  * Once per hour Superblock Submitters create a new Superblock containing the newly created blocks and send a Superblock summary to [SyscoinClaimManager contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinClaimManager.sol)
  * Superblock Challengers will challenge the superblock if they find it invalid. They will request the list of block hashes, the block headers, etc. Superblock Submitters should send that information which is validated onchain by the contract.
  * If any information provided by the Superblock Submitter is proven wrong or if it fails to answer, the Superblock is discarded.
  * If no challenge to the Superblock was done after a contest period (or if the challenges failed) the superblock is considered to be "approved". [SyscoinClaimManager contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinClaimManager.sol) contract notifies [SyscoinSuperblocks contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinSuperblocks.sol) which adds the Superblock to its Superblock chain.
  * Note: [SyscoinSuperblocks contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinSuperblocks.sol) uses a checkpoint instead of starting from Syscoin blockchain genesis.
  * Note: [Sysethereum-Dapp](https://github.com/syscoin/sysethereum-dapp) was created with this workflow in mind in an automated native ReactJS application for convenience.

* Sending Syscoins to ethereum
  * User creates burn sys tx on the sys network using by calling `syscoinburn` or `assetallocationburn`.
  * The sys tx is included in a sys block and several sys blocks are mined on top of it.
  * Once the sys block is included in an approved superblock, the burn tx is ready to be relayed to the eth network.
  * The use sends an eth tx to [SyscoinSuperblocks contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinSuperblocks.sol) containing: the sys burn tx, a partial merkle tree proving the sys burn tx was included in a sys block, the sys block header that contains the sys burn tx, another partial merkle tree proving the block was included in a superblock and the superblock id that contains the block.
  * [SyscoinSuperblocks contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/SyscoinSuperblocks.sol) checks the consistency of the supplied information and relays the sys burn tx to [SyscoinToken contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/token/SyscoinToken.sol).
  * [SyscoinToken contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/token/SyscoinToken.sol) mints N sysx tokens and assigns them to the User. Syscoin burn txs specify a destination eth address.


* Sending sysx tokens back to syscoin
  * User sends an eth tx to the [SyscoinToken contract](https://github.com/syscoin/sysethereum-contracts/blob/master/contracts/token/SyscoinToken.sol) invoking the `burn` function. Destination sys address, amount and asset id are supplied as parameters.
  * The User after waiting 240 confirmations on Ethereum creates, signs & broadcasts a sys mint tx using `syscoinmint` or  `assetallocationmint` depending if moving Syscoin or an asset on syscoin. 
  * The user receives the unlocked sys.

## Incentives

Some operations require gas to be spent or eth deposit to be frozen. Here are the incentives for doing that.

* Submitting a Superblock: Superblock submitters will get a fee when the superblock they sent is used to relay a tx. The Agent may connect to the local Geth node running alongside Syscoin. The account on the Geth node used by the Agent (configured in the conf file in the Agent settings) should be set to this account and unlocked on the Geth node.
* Superblock challenge: Challengers who find invalid superblocks will get some eth after the challenge/response game finishes.
* Each user will Syscoin txs to SyscoinSuperblock to go to Ethereum or calling the Burn function on the SyscoinToken to go back to Syscoin. The user is also responsible for paying the Syscoin network fees for creating burn and mint transactions on the Syscoin network.


## Assumptions
* Incentives will guarantee that there is always at least one honest Superblock Submitter and one honest Superblock Challenger online.
* There are no huge reorgs (i.e. 100+ block) in Sys nor in Eth blockchains

## Team

* [Jagdeep Sidhu](https://github.com/sidhujag)
* [Willy Ko](https://github.com/willyko)
* [Dan Wasyluk](https://github.com/dwasyluk)

## License

MIT License<br/>
Copyright (c) 2019 Jagdeep Sidhu <br/>
Copyright (c) 2018 Coinfabrik & Oscar Guindzberg<br/>
[License](LICENSE)
