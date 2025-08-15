
# Attacker can steal user funds by using reward mechanism

## Summary
Attacker can directly transfer the USDT to the contract to call the, `subDepositReward` to inflate the `virtualTokens` and steal initial depositor's fund

## Root cause
`PortfolioToken.sol` used by the users to deposit and withdraw the USDT tokens, they can do this by selecting one of the 4 indexes in the contract, which will deposit the token as LP in  respective pool
However, the contract provide the following function 
[Here](https://github.com/sherlock-audit/2025-07-allbridge-core-yield/blob/a14f35e01b9bf8f4e5ad2b5ce1ff61dd59941e16/core-auto-evm-contracts/contracts/PortfolioToken.sol#L142-L150)

Which allows anyone to call this function and submit the reward in USDT already present in the contract 
by this, the internal function [here](https://github.com/sherlock-audit/2025-07-allbridge-core-yield/blob/a14f35e01b9bf8f4e5ad2b5ce1ff61dd59941e16/core-auto-evm-contracts/contracts/PortfolioToken.sol#L152-L162) is triggered where this **RE-DEPOSITS the funds, without minting extra shares for it**
this can allow the attacker to inflate the initial deposits and steal it. (in the next section)

## Internal Pre-condition
No deposits done yet or everything is back to its original state (x amount deposited, x amount withdraw instantly)

## External-Pre conditions
None

## Attack path
**Note** -> This attack require some amount of funds which can be taken a flash loan easily

1. Attacker transfers 1000 USDT (can be more or less) directly to the account
2. calls `subDepositRewards` (this is optional because this function is already called in `deposit()`)
3. calls `Deposit()` with params `virtualAmount` = 1 wei and any admin set index 

At this point the `realtotal`  = 1 wei `totalVirtualAmount` = 1000e6
and that 1wei  is minted to the Attacker
4. Now if the legit user with legit initial deposit tries to deposit any amount less than 1000 USDT token, the net shares minted to him would be **ZERO** because of [this](https://github.com/sherlock-audit/2025-07-allbridge-core-yield/blob/a14f35e01b9bf8f4e5ad2b5ce1ff61dd59941e16/core-auto-evm-contracts/contracts/VirtualMultiToken.sol#L191-L208) calculation
At this point, minted shares would still me 1 wei and total virtual amount would be 1000 USDT +
5. Now the Attacker can withdraw that amount, calling the `withdraw()`, that way he will get 
1000 USDT + User initial Deposit + reward(if any)

## Impact
The user will lose all of funds given to this contract, The attacker can repeat the steps again and again for the different user.
Hence, loss of user funds

## POC 
N/A

## Mitigation
Some amount of dead shares should be minted while deploying the address to the  same contract, this was it will be less likely. If possible there should also be some amount of MIN_DEPOSIT.
