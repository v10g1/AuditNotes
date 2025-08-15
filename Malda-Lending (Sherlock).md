
This audit is based on a lending protocol, which is going to be deployed on a RISC zero ZK-VM

## Claiming of Malda tokens
Below is the explainer of the full process of claiming tokens in Malda protocol 
everything starts with the explainer function like `ClaimMalda()` or something similar to  
[This](https://github.com/sherlock-audit/2025-07-malda/blob/4618eca0f09a2e643008f43f12a2fb8206975a3b/malda-lending/src/Operator/Operator.sol#L516)
Which later calls 
[this](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/Operator/Operator.sol#L844)
 
This has two options, first one which allows the distribution of the borrower funds and second one if the suppliers flag is true
1. Borrower can
for each token that the holder holds, this function should be called for, with will update the borrow index and discard the deprecated indexes
```solidity
_updateMaldaBorrowIndex(address(mToken));
```
which in turn calls  `_notifyBorrowIndex` for each reward token a `mtoken` has 

**High Level**-> from what i can see is that this protocol will have multiple lending markets, which will have one `mtoken` per market and then multiple reward tokens per `mtoken` , which means reward token for a singular lending market can be more than One (or even one)

One thing to note in the `_notifyBorrowIndex()` is that 
```
uint256 deltaBlocks = blockTimestamp - marketState.borrowBlock;
```
Which means, `deltaBlocks` is depend whenever the `marketState.borrowBlock` was set, which was set in this same transaction.
This means if someone to calls the function twice in the same function it will just return **ZERO**

Question -> Can this be somehow manipulated in attacker's favor?

we can see [here](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/rewards/RewardDistributor.sol#L272-L278)
that we do `borrowIndex` + ratio, in which ratio = accrue / total borrow

and accrue = `deltablocks` +`borrowSpeed`
 Answer-> discuss later
After this function, `_claim()` calls [`_notifyBorrower`](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/rewards/RewardDistributor.sol#L313)

This takes the borrow index which we found earlier, was reset in every call. After which, the `_grantReward` function is used to actually transfer the funds from contract to the user with a custom transfer logic.

## Custom transfer logic in `Mtokens`
it uses some kind of custom hooks to facilitate the proper transfer of the function,
There are four functions which are called during the pretransfer of the funds 
`_beforeRedeem()`
`_updateMaldaSupplyIndex()`
`_distributeSupplierMalda()`
`_distributeSupplierMalda()`

1. `beforeRedeem()` -> minor check (paused and existence of market)
this function has a MAJOR check for skipping the liquidity check 
which goes as follows
```solidity
if (!markets[mToken].accountMembership[redeemer]) return;
```
Here it says that if the account DOES NOT have the membership of the market, then it will return without calling the other line which checks the liquidity
```solidity
(, uint256 shortfall) = _getHypotheticalAccountLiquidity(redeemer, mToken, redeemTokens, 0);
```
This function calculates the shortfall which will be acquired AFTER (if) the redemption went through, it is extremely important check for protocol insolvency 

## Msg sending across the chain in Malda
Malda protocol allows the rebalancing of the markets with a single click across all chains  
for this they use the function `sendMsg` in `Rebalancer.sol`. to call this function the `msg.sender` should be `REBALANCE_EOA` the bridge should be whitelisted, and the `dstChain` chain should also be whietlisted, 
-> It weirdly assumes that the market on the chain is Not paused. can this be somehow exploited?
It should also exceed minimum transfer size of that `DST` chain
it has the following check for the respective chain
```solidity
TransferInfo memory transferInfo = currentTransferSize[_msg.dstChainId][_msg.token];

        uint256 transferSizeDeadline = transferInfo.timestamp + transferTimeWindow;

        if (transferSizeDeadline < block.timestamp) {

            currentTransferSize[_msg.dstChainId][_msg.token] = TransferInfo(_amount, block.timestamp);

        } else {

            currentTransferSize[_msg.dstChainId][_msg.token].size += _amount;

        }
```
Here we can see, everytime a new msg is sent the transfer Deadline is extended by the `transferwindow` .

Now if the `currentTransferDeadline` is less than `block.timestamp` we completely replace the transfer req with a new one

However when we see the follow line of code 
```solidity
uint256 _maxTransferSize = maxTransferSizes[_msg.dstChainId][_msg.token];

        if (_maxTransferSize > 0) {

            require(transferInfo.size + _amount < _maxTransferSize, Rebalancer_TransferSizeExcedeed());

        }
```

it has a MAJOR check for reverting, which is `required` statement, it says that if the `transferInfo.size` + amount is more than `_maxTransferAmount` then it should revert.

However there is a major flaw is this logic, the `transderInfo.size` takes the value BEFORE the changes in the if-else block was initiated, specifically when the `transferSizeDeadline`  < `block.timestamp` it considers a case where it completely replaces the previous request of `transferSize` . In this case, the `currentTransferSize` is already equal to `amount` and still adds `_amount` to it, which may cause a permanent DOS, to this function 

Example values
before transfer block -> 100
`transferDeadLine` -> 10 blocks
before transfer amount 1000e18 `wETH`
`maxSize` -> 1500e18 `wEth`
new transfer amount = 800e18 `wETH`
current block ->100+15 blocks = 115 
at this state the `if` statement will be triggered which will over write the `currentTransferSize` from 1000e18 to 800e18 
**BUT** `transferInfo.size` will still be equal to 1000e18 

Now at the point of this check the condition 
`transferInfo.size + _amount < _maxTransferSize` will be given as
1000e18 +800e18 < 1500e18

which is **FALSE** hence it will just revert and cause DOS, 

## Bridge implementation in `Malda-lending`
The bridges are used to rebalance liquidity in the markets across all chain, if a market on chain1 has more liquidity then it can be donated to same type of market on other chains, this will help maintain the overall cycle of the market.
There are two bridge protocol used in `malda` which are 
1. `across`
2. `EverClear`
#### `Sendmsg` function in Across bridge

- First check -> 
```solidity
require(_extractedAmount == msgData.inputAmount, BaseBridge_AmountMismatch());
```
`_extractedAmount` is the parameter passed from rebalancer and other value is the decoded value from the msg param. 

```solidity
require( 
msgData.inputAmount - msgData.outputAmount <= maxSlippageInputAmount, AcrossBridge_SlippageNotValid()
);
```
This is the slippage check for the bridge Out tokens with the respective In-token.

and then it calls the bridging function.

#### `sendMsg` function in Ever clear bridge

This function is also called by the rebalancer contract from a semi trusted EOA.

`bytesLib` is OOS so we will consider that this is working just fine.

There is a check as 
```
        require(_extractedAmount >= params.amount, BaseBridge_AmountMismatch());
```
Here, `_extractedamount` is the amount transferred to the rebalancer from the market  
in the later part of the same function we see that in same the function , the excess amount is sent back 
### `MToken` implementation in `malda`

From what i can see, The contract use other abstract contracts for the implementations of external and internal functions
The explanation of the code is in the increasing order of the SNLOC on the `mtoken` contracts. 

`mERC20immutable.sol`
-> Implements a constructor which handles the `init` process of the the abstract contract
Which in our case, is `mERC20`
The admin is set to the `msg.sender` of this contract 
`mERC20upgradable.sol`
It is an abstract contract
Implements the same type of contract with the core difference comming along in that way that 
```solidity
 constructor() {
        _disableInitializers();
}
```
`_disableInitializers()` is thee function from OZ, from the INIT lib

defined as 

```solidity
function _disableInitializers() internal virtual {

InitializableStorage storage $ = _getInitializableStorage();
if ($._initializing) {
revert InvalidInitialization();
}
if ($._initialized != type(uint64).max) {
$._initialized = type(uint64).max;
emit Initialized(type(uint64).max);
}
}
```
here we can clearly see that this disables the contract's ability to 
this disallows the initialization of the other inherited contracts. 

`mtokenConfiguration`

This Abstract piece of code allows the controlling and management of token essential parameters
To add any parameter the caller must be an admin
The changes include 
Set operator
Set Roles Operator
Set Interest rate model
set `BorrowRateMaxMantissa`
etc.

These were the opening contracts used by the `Mtoken`


The Detailed Implementation of the same :
The `mERC20Host.sol` inherits `mERC20upgradable.sol` which inherit  `mERC20.sol` further inherits `mToken.sol` 



## Risc-Zero VM

We can create ZK co-processors which can then compute the expensive part of the application off-chain 
Let's us prove the correct execution of arbitrary rust code.

1. the guest program is compiles to in ELF binary 

Guest program -> be able to read inputs, write private outputs to the host (the system that zkVM runs on) and commit public outputs to the journal(portion of receipt that contains the public outputs of a zkVM app)

The guest program should not have 
-> `No_std`
-> `no_main`
-> `risc0_zkvm_guest::entry!(main)`
This is to make the module as less heavy as possible to make it faster

ELF Binary -> like solidity compiles into the lower level languages like `yul` the ELF BINARY is the executable format of the Rust code written. 
2. The executor runs the ELF binary and records the session
3. the prover checks and proves the validity of the session, outputting the receipt![[Pasted image 20250803101434.png]]
this image shows the execution of the whole environment in RISC zero
A very simple host program can be written as 
```rust
use risc0_zkvm::{default_prover, ExecutorEnv};

let env = ExecutorEnv::builder().build().unwrap();
let prover = default_prover();
let receipt = prover.prove(env, METHOD_NAME_ELF).unwrap().receipt;
```
When we look into this code we can see that we need to import the dependencies to the rust environment , The env is necessary to transfer the necessary info the the prover, which in turn can make the receipt for the user 

Now to actually verify the proof, the verifier needs to call the `verify` function as 

```rust
reciept.verify(METHOD_NAME_ID).unwrap();
```

Methods name id is made during the compilation, which in turn, by the user, needs to be imported to the contract.
This will be discussed more in the individual part.

## Working of Abi.decode encode etc

These function allows you to convert solidity data types like string, integers, address, etc into a compact binary representation to back to the normal data type

**ABI.encode** -> Abi is Application binary interface. It enocdes multiple value in a tighly packed

decode -> it unpacks the packed bytes to desired data types

**However** it can lead to the function to internally revert

**Let's discuss how**
https://medium.com/@0xdeadbeef0x/the-double-edged-sword-of-abi-decode-f81529e62bcc

^^ credits, io10-0x github.

I got this resource when i was trying to read the audit notes of other cool auditors and their finding, this resource was made by 0xdeadbeef.

Malda uses ab.decode on multiple instances where Risc zero proofs are decoded from the data to the bytes, i have started to read somepart of the Risc zero implementation which can be seen aabove this section 

We know that the reverts should not happen when we are talking about zero knowledge proofs, when the verifier is trying to verify the proof.

anyway, let's see how the abi.decode works under the hood.

Solidity has released a [blog](https://blog.soliditylang.org/2021/04/21/custom-errors/) post detailing how to efficiently revert with a custom reason.

```
let free_mem_ptr := mload(64)  
mstore(free_mem_ptr, 0x08c379a000000000000000000000000000000000000000000000000000000000)  
mstore(add(free_mem_ptr, 4), 32)  
mstore(add(free_mem_ptr, 36), 12)  
mstore(add(free_mem_ptr, 68), "Unauthorized")  
revert(free_mem_ptr, 100)
```

Here we can see that we load the 0x40 which is the free memory pointer in solidity 
mload loads word from the memory, in input stack we give the offset in bytes and 
in output we get **32 bytes** in memory starting at that offset. If its beyond current size 
So in our case we get 32 bytes extra after the free memory pointer
It then selectes the function selector of "Error(String)" at free_mem_ptr

The rest packed with zeros. Why? because we need it to be padded with 32 bytes

Then it writes 0x20 offset at the free_mem_ptr+4, because the first 4 bytes are filled already, otherwise it will just overrite it and the previously stored value will be gone

It then stores the length of the string which is 12 bytes(decimals), 

At offset 0x44 (`free_mem_ptr + 68`), you store the **string bytes** `"Unauthorized"`.

and at the end, it reverts the transaction, returning 68+32 bytes from the free_memory_point
**PARSING THE REVERT REASON**
```solidity
(bool success, bytes memory result) = address(target).call(data);  
  
if (!success) {  
// Next 5 lines from https://ethereum.stackexchange.com/a/83577  
if (result.length < 68) revert();  
assembly {  
result := add(result, 0x04)  
}  
revert(abi.decode(result, (string)));  
}
```
This line of code is used in protocols like Uniswap V3

In the snippet above, the data returned by the low-level call is pointed to in the "results" and the code adds 4 butes to the offset of the result pointer, You cannot just simply add or remove the 4 bytes.



The network is said to be deployed on ETH, unichain, linea, Base, and OP
The base and Op chain uses the Op stack.
Malda uses RISC zkVm for multiple cross chain verification and communication 

## Op Stack

**OP Stack chains (Base, Optimism)** don’t store _L2 state_ directly on-chain like normal L1 chains. Instead:

- They post **L2 output roots** to **Ethereum L1**.
 
- These outputs are published by a special role called the **Proposer** (or Sequencer).
 
- The actual **block headers, state roots, and storage proofs** of L2 aren’t easily accessible **on the L2 chain itself** — because L2s don’t keep their own historical data like Ethereum L1 does.
Which means, That we need to QUERY the L1 contracts instead of an L2

What is an RLP Header
an **RLP header** typically refers to the **RLP-encoded block header**. Let’s break this down:

**RLP** stands for **Recursive Length Prefix**. It's Ethereum's core serialization method used to encode nested arrays of binary data (like block headers, transactions, and receipts) into a compact byte format.

- It's not human-readable.
    
- It allows variable-length data to be packed and parsed efficiently.
    
- It's used to encode most data structures that go into the blockchain.
So in a nutshell it is the summary(metadata) of the block.

```
f90211a08368d289725ec6b68c3f7501bbf99e5f4cd254f4e11d26d29df1816b05e1209801a01dcc4de8dec75d7aab85b567b6cc77f2a47fd4...

```
This is a **truncated** version of the full RLP blob for brevity. Full RLP hex is usually ~500–800 bytes long.

The data would be something like this 

![[Pasted image 20250807153021.png]]
 **Differences between OP stack and ETHEREUM**
 OP Stack chains are designed to be [EVM equivalent](https://web.archive.org/web/20231127160757/https://medium.com/ethereum-optimism/introducing-evm-equivalence-5c2021deb306) and introduces as few changes as possible to the Ethereum protocol. However, there are some minor differences between the behavior of Ethereum and OP Stack chains that developers should be aware of.

**Bridging**
Deposit transactions don't exist on L1s, and are how transactions on an L2 can be initiated from the L1. Importantly, this is how bridge applications can get L1 ETH or tokens into an L2 OP Stack chain.

Information is encapsulated in lower layer packets on the sending side and then retrieved and used by those layers on the receiving side while going up the stack to the receiving application.


## Game Theory in Optimistic chains
Optimism is an **Optimistic Rollup** — it batches transactions off-chain and posts the results to Ethereum L1.
The word “optimistic” means:

- The system assumes submitted L2 state roots are correct **by default**. (Optimistic)
    
- Anyone can challenge a posted state if they think it’s wrong. they have cetain time period for it, for OP its 7 days per proof
That challenge process is structured like a **game** between two parties:

- **Proposer (challenged party)** — claims the state is correct.
    
- **Challenger** — claims it’s incorrect.
Optimism’s new **Fault Proofs** framework defines each verification challenge as a **dispute game** on Ethereum L1.  
There are **multiple types** of games (modular system):

1. **Output Dispute Game (ODG)**
    
    - Checks _final L2 state outputs_ (state roots) posted to L1.
        
    - If someone disputes, the game tries to prove or disprove that the posted output is correct.
        
2. **Execution Dispute Game (EDG)**
    
    - Re-runs the disputed L2 transactions to see if they produce the claimed output.
        
    - Like “replaying the tape” of what happened.
        
3. **Other games (potential future)**
    
    - OP Stack is modular — more specialized verification games could be added.
How does it work??
Imagine it like a **binary search game** on the transaction sequence:

4. **Claim submission**
    
    - The proposer posts a claim: “Here’s the L2 state root after block #12345.”
        
5. **Challenge**
    
    - A challenger says: “I think your claim is wrong.”
        
6. **Step-by-step narrowing**
    
    - Both parties present intermediate state claims at specific points in the transaction sequence.
        
    - This continues until they find the **exact transaction** where they disagree.
        
7. **On-chain execution**
    
    - That one transaction is re-executed **on Ethereum L1** to decide the truth.
        
8. **Resolution**
    
    - If proposer lied → their bond is slashed, challenger rewarded, claim rejected.
        
    - If challenger lied → challenger’s bond is slashed, proposer rewarded, claim stands.

## What are Snarks and Starks
**Both are types of Zero-Knowledge Proofs (ZKPs)** — cryptographic proofs that let you prove you know some secret or performed some computation _without revealing_ the secret or the full computation itself.

- **SNARK** = _Succinct Non-interactive Argument of Knowledge_
 
- **STARK** = _Scalable Transparent Argument of Knowledge_
