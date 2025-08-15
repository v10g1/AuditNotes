Title -> Incorrect check in `rebalancer.sol` causes DOSes the `sendMSG` function 

Summary

incorrect check for `_maxTransferSize` in [`rebalancer.sol`](https://github.com/sherlock-audit/2025-07-malda/blob/main/malda-lending/src/rebalancer/Rebalancer.sol) causes DOS in the `sendMsg` function, which disables the cross chain messaging

Root Cause

We can see [here](https://github.com/sherlock-audit/2025-07-malda/blob/4618eca0f09a2e643008f43f12a2fb8206975a3b/malda-lending/src/rebalancer/Rebalancer.sol#L141-L152) the function tries two things to do 
- Checks if the pervious order deadline was met 
- checks thatg if the **Total** size of the transfer is less than the `_maxTransferSize` which is the cap of transfer amount **PER ORDER**
However we see below 
```Solidity
TransferInfo memory transferInfo = currentTransferSize[_msg.dstChainId][_msg.token];
        uint256 transferSizeDeadline = transferInfo.timestamp + transferTimeWindow;
        if (transferSizeDeadline < block.timestamp) {
            currentTransferSize[_msg.dstChainId][_msg.token] = TransferInfo(_amount, block.timestamp);
        } else {
            currentTransferSize[_msg.dstChainId][_msg.token].size += _amount;
        }

        uint256 _maxTransferSize = maxTransferSizes[_msg.dstChainId][_msg.token];
        if (_maxTransferSize > 0) {
            require(transferInfo.size + _amount < _maxTransferSize, Rebalancer_TransferSizeExcedeed());
        }

```
the transferInfo is stored **BEFORE** the updates happens in the if-else block
Now if the `transferSizeDeadline` is passed for the msg before (Meaning it had already been executed), this replaces the previous request with a new struct.

However, when the check for `_maxTransferSize` (assuming it is already set) tries to check the condition, we see 
```solidity
require(transferInfo.size + _amount < _maxTransferSize, Rebalancer_TransferSizeExcedeed());
```
that the **PREVIOUS** `transferInfo` is also accounted for, which in that case would have been changed.

This can cause the revert if the `_amount` + `TransferInfo.size` is **MORE** than `_maxTransferSize`, which will be false because `TransferInfo.size` is already changed to amount in the previous if-else statment.

Internal-pre condition 
-> There exists atleast one **executed** msg from the rebalancer before (meaning the transferSizeDeadline for that has been already crossed)

External-Pre condition
None

Attack path 
Let's say that the 
previous `TransferInfo.size` = 1000e18 units
previous `TransferInfo.timestamp` = 100 
TransferTimeWindow = 10 
`_maxTransferSize` =1500e18
Now currently
timestamp = 200
in this function the `REBALANCER_EOA` (semi-trusted EOA) calls the `sendmsg()` with 
`_amount` 800e18 (let's say)

Now because that the previous msg has already been executed system implement `if` statement where the previous (`TransferInfo.size` = 1000e18 units) gets replaced with the new one, which is 800e18

However the check
```solidity
require(transferInfo.size + _amount < _maxTransferSize, Rebalancer_TransferSizeExcedeed());
```
considers `transferInfo.size` =1000e18 and `_amount` =800e18
that makes the condition 
`1000e18+800e18 <1500e18`
Which is clearly false and this transaction would revert.

Impact -> 
The rebalance msg won't be delivered and other chain pool will still be unbalanced

Poc 
None

Mitigation-> 
Updating the `TransferInfo` again after the `if` block.

---

Title-> DOS for cross-chain messaging in UNICHAIN and ARBITRUM chains

Summary 
The transaction (cross-chain message) triggers a panic when the `chain_id` is not one of the following:
- `ETH_MAINNET`
- `OPTIMISM`
- `Linea`
- `BASE`

However, it crucially **omits support for `ARBITRUM` (chain ID: 42161)** and **`UNICHAIN` (chain ID: 130)**. This assumption forgets the access on arbitrum and unichain
Root cause ->
According to the [readme](https://github.com/sherlock-audit/2025-07-malda/blob/main/README.md) The protocol is supposed to be deployed in the chains
- Eth mainnet
- OPTIMISM
- LINEA
- BASE
- ARBITRUM
- UNICHAIN (by uniswap)
But when we see the following piece of code 
```solidity
  if chain_id != LINEA_CHAIN_ID && chain_id != BASE_CHAIN_ID && chain_id != ETHEREUM_CHAIN_ID && chain_id != OPTIMISM_CHAIN_ID {
            panic!("Chain ID is not Linea, Base, Ethereum or Optimism");
        }

        // This makes the guest program only compatible with testnet chains, remove for mainnet and enable the above mainnet code
        // if chain_id != LINEA_SEPOLIA_CHAIN_ID && chain_id != BASE_SEPOLIA_CHAIN_ID && chain_id != ETHEREUM_SEPOLIA_CHAIN_ID && chain_id != OPTIMISM_SEPOLIA_CHAIN_ID {
        //     panic!("Chain ID is not Linea Sepolia, Base Sepolia, Ethereum Sepolia or Optimism Sepolia");
        // }
        
        validate_get_proof_data_call(chain_id, account, asset, target_chain_ids, env_input, sequencer_commitment, env_op_input, &linking_blocks, &mut output, &env_eth_input, op_evm_input, sequencer_commitment_opstack_2, env_op_input_2);
    }
```
Here we can see that there is no consideration that the `chain_id` (which in our case is SRC chain)being the Arbitrum (ID =42161) and UNICHAIN (ID=130) for being one of the src chain. This will revert all the cross chain requests for these chains.

Internal-pre condition
None

External pre condition
None

Attack Path 
Any requests from UNICHAIN and ARBITRUM would revert.
No specfic attack but this DOSes the cross chain messaging if we are trying to rebalance/hosting from these chain.

Impact->
The core functions(Like rebalancing cross chain) will not be able to perform as intended

Poc 
none

Mitigation->
Add the support for the chains ie UNICHAIN(id=130) and ARBITRUM(id =42161)

---

title ->Borrow Rate Max Bypass for Operations Within the Same Block

Summary 
For consecutive transactions in a single block the `BorrowRateMantissa` can exceed `MaxBorrowRateMantissa` . By-passing the crucial check


root cause ->
when we see the following `accureinterest()` function
```solidity
function _accrueInterest() internal {
        /* Remember the initial block timestamp */
        uint256 currentBlockTimestamp = _getBlockTimestamp();
        uint256 accrualBlockTimestampPrior = accrualBlockTimestamp;

        /* Short-circuit accumulating 0 interest */
        if (accrualBlockTimestampPrior == currentBlockTimestamp) return;

		...

        uint256 borrowRateMantissa =
            IInterestRateModel(interestRateModel).getBorrowRate(cashPrior, borrowsPrior, reservesPrior);
        if (borrowRateMaxMantissa > 0) {
            require(borrowRateMantissa <= borrowRateMaxMantissa, mt_BorrowRateTooHigh());
        }

    .....

    
    }
```

The function performs two main checks. First, it verifies whether interest was accrued at the same timestamp. If so, no further validation is required, and the function returns immediately. However, after this initial check, there is an important validation for `MaxBorrowRateMantissa`, ensuring that the current borrow rate is less than the allowed maximum. When the interest is accrued within the same block, this second check is skipped for the second (or multiple) transactions, potentially allowing a transaction to bypass the borrow rate limit for the latter requests.

Internal pre-condition
None

External Pre-condition 
None

Attack Path

1. In the first transaction of a block, `accrueInterest()` is called and the timestamp gets updated.  
2. After that, market conditions becomes worse, (May have caused because of that first transaction).  
3. Another transaction in the *same* block calls `accrueInterest()` again, but it returns early because the timestamp didn’t change, so the borrow rate check never runs.    
4. This means the tx is executed even though it’s above rate limits.  
5. If you try the exact same tx in the next block, it would revert, which makes protocol behavior inconsistent.
Impact 
- Borrowing can still be done when the rate fo interests are more than the MaxBorrowRateMantissa
Poc
None

Mitigation  
Moving the check before the the timestamp check should fix this issue.

___
title->
Incorrect Modifier Allows Blacklisted Users to Evade Liquidation

Summary->
Modifier misplacement in [`Operator.sol::beforeMTokenLiquidate()`](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/Operator/Operator.sol#L670-L692) and [`Opersator.sol::beforeMTokenSeize()`](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/Operator/Operator.sol#L697-L715) allows blacklisted users to evade liquidation despite shortfall conditions. 

Root Cause -> 
The `beforeMTokenLiquidate()` function in `Operator.sol` determines whether a borrower is eligible for liquidation. 
If a borrower meets the eligibility criteria, the liquidation process should not revert. 

However, the `ifNotBlacklisted(borrower)` modifier on this function checks whether the borrower is blacklisted:

```solidity
modifier ifNotBlacklisted(address user) {
    require(!blacklistOperator.isBlacklisted(user), Operator_UserBlacklisted());
    _;
}

```
This means that if a borrower is underwater while being blacklisted, no one will be able to liquidate their position.  
As a result, the position may remain insolvent, and if asset prices drop further, the shortfall could increase,  
leading to greater protocol losses.

internal pre-condition 
Blacklisted User with (some) debt

External pre-condition 
none

Attack path:
1. User Bob enters the market and borrows a certain amount.  
2. Later, Bob is blacklisted.  
3. His position in the MToken becomes eligible for liquidation.  
4. The liquidation transaction reverts, allowing the position’s value to deteriorate and causing a loss to the protocol.
Impact ->
Tbe blacklisted user can evade the liquidation via this.
Mitigation 
the removal of the modifier in both of the function, should fix this issue

title->
Funds May Become Trapped During Migration from Mendi to Malda Markets
Summary 
If the amount of collateral received upon redemption exceeds the position’s collateral in the Mendi market, the surplus funds may become trapped within the contract, with no available mechanism to withdraw them.

RootCause
`Migrator.sol` helps move positions from V1 (Mendi) to V2 (Malda). It works by first minting the same amount of collateral for the user (`msg.sender`) in the Malda markets. Then, it borrows against that collateral and repays the loan in the Mendi market. But there’s a problem when the code tries to withdraw collateral from Mendi and transfers it to Malda, as you can see below.
```solidity
        for (uint256 i; i < posLength; ++i) {
            Position memory position = positions[i];
            if (position.collateralUnderlyingAmount > 0) {
                uint256 v1CTokenBalance = IMendiMarket(position.mendiMarket).balanceOf(msg.sender);
                IERC20(position.mendiMarket).safeTransferFrom(msg.sender, address(this), v1CTokenBalance);


                IERC20 underlying = IERC20(IMendiMarket(position.mendiMarket).underlying());


                uint256 underlyingBalanceBefore = underlying.balanceOf(address(this));


                // Withdraw from v1
                // we use address(this) here as cTokens were transferred above
                uint256 v1Balance = IMendiMarket(position.mendiMarket).balanceOfUnderlying(address(this));
                require(
                    IMendiMarket(position.mendiMarket).redeemUnderlying(v1Balance) == 0,
                    "[Migrator] Mendi withdraw failed"
                );


                uint256 underlyingBalanceAfter = underlying.balanceOf(address(this));
                require(
                    underlyingBalanceAfter - underlyingBalanceBefore >= v1Balance, "[Migrator] Redeem amount not valid"
                );


                // Transfer to v2
@>               underlying.safeTransfer(position.maldaMarket, position.collateralUnderlyingAmount);// Amount Transfered
            }
        }
```

The code first transfers all Mtokens (MENDI tokens) from the user to this contract, and then withdraws them to repay the debt.

This step ensures that the actual assets are transferred to the Malda market, as executed in the final line of the code.

Any migration or other operations within the Mendi market can reduce the interest rate in the Mendi pool, which may result in a higher amount of collateral available during withdrawal.

However, if there is a discrepancy between the amount transferred and the position collateral, the excess funds beyond the position collateral will remain trapped within the contract. This occurs because there is no mechanism to withdraw these surplus funds, as indicated by the following line.
```solidity
uint256 underlyingBalanceAfter = underlying.balanceOf(address(this));
                require(
                    underlyingBalanceAfter - underlyingBalanceBefore >= v1Balance, "[Migrator] Redeem amount not valid"
                );


                // Transfer to v2
@>               underlying.safeTransfer(position.maldaMarket, position.collateralUnderlyingAmount);
```

which only transfers the position collateral to the Malda market. Any excess amount—calculated as `(underlyingBalanceAfter - underlyingBalanceBefore) - PositionCollateral`—will remain permanently locked within the contract.

Internal pre-condition 
none
external pre-condition 
none 

Attack Path 
1. Bob has called `MigrateAllPositions` in `migrator.sol`
2. His positions are migrated 
3. Change in Interest rate would result in getting slightly more collateral than intended
4. This amount Will be stuck in the `Migrator.sol` contract.

Impact 
The residual amount of token from the withdraw can be permanently stuck in this(`Migrator.sol`) contract

Mitigation 
Handling of this residual funds via eigther
1. Transfering the remaining amount back to the user
2. Adding a restricted function for admin to withdraw these funds.

---

title->
Incorrect `minAmount` Parameter Can Cause Denial-of-Service in Migration Operations
Summary 
An incorrect `minAmount` parameter passed to the `_mint` function in `migrator.sol::migrateAllPositions` can lead to a complete denial-of-service (DoS) for migrations from the Mendi market to the Malda market.

Root cause->
The `migrator.sol` contract is responsible for migrating positions from the Mendi market to the Malda market.  
As part of this process, the contract first mints the same amount of `underlyingCollateral` in the Malda market that exists in the Mendi market. This is done as follows:

```solidity
        for (uint256 i; i < posLength; ++i) {
            Position memory position = positions[i];
            if (position.collateralUnderlyingAmount > 0) {
                uint256 minCollateral =
                    position.collateralUnderlyingAmount - (position.collateralUnderlyingAmount * 1e4 / 1e5);
                ImErc20Host(position.maldaMarket).mintOrBorrowMigration(
                    true, position.collateralUnderlyingAmount, msg.sender, address(0), minCollateral
                );
            }
        }
```
which is [here](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/migration/Migrator.sol#L91-L100) 
Here, if the collateralUnderlying amount in the respective Mendi market is greater than zero, we consider it for migration.

Next, the contract calculate the minimum amount of underlying collateral that should be received. This works as a safety mechanism to protect against frontrunning and sandwich attacks.

However, the code is actually passing the **minimum amount of underlying** we need (set to 90% of the underlying in the Mendi market).

Let’s take a closer look what is being called here.

```solidity
function mintOrBorrowMigration(bool mint, uint256 amount, address receiver, address borrower, uint256 minAmount)
        external
        onlyMigrator
    {
        require(amount > 0, mErc20Host_AmountNotValid());

        if (mint) {
            _mint(receiver, receiver, amount, minAmount, false);
            emit mErc20Host_MintMigration(receiver, amount);
        } else {
            _borrowWithReceiver(borrower, receiver, amount);
            emit mErc20Host_BorrowMigration(borrower, amount);
        }
    }
```
we call this function which is [here](https://github.com/sherlock-audit/2025-07-malda/blob/798d00b879b8412ca4049ba09dba5ae42464cfe7/malda-lending/src/mToken/host/mErc20Host.sol#L305C5-L318C6) 
Here, if the bool is true (which in our case, it is), we call the `_mint` function. This function is inherited from `mToken._mint`, which later calls `__mint()`.

Let’s now investigate that part in more detail.
```javascript
    function __mint(address minter, address receiver, uint256 mintAmount, uint256 minAmountOut, bool doTransfer)
        private
    {
        IOperatorDefender(operator).beforeMTokenMint(address(this), minter);


        Exp memory exchangeRate = Exp({mantissa: _exchangeRateStored()});


        
        uint256 actualMintAmount = doTransfer ? _doTransferIn(minter, mintAmount) : mintAmount;
        totalUnderlying += actualMintAmount;


        


        uint256 mintTokens = div_(actualMintAmount, exchangeRate);
        require(mintTokens >= minAmountOut, mt_MinAmountNotValid());


        
        //.......
    }
```
Here we can see that the function try to check for the slippage.

**However**, it is checking the slippage on the minted token, not on the underlying it is actually accounting for.

As we saw earlier, the migrator is passing the **underlying amount it needs** rather than the **minted token amount**.

This is clearly wrong and can cause a permanent DoS once the exchange rate changes (which happens quite often after borrows). When 1 minted token != 1 underlying token, this can result in a DoS.

Internal pre-condition
none

External pre-condition 
none

Attack path 
1. Bob has a position in mendi market
2. He tries to migrate it 
3. The migration fails due to incorrect params (as explained above) 

Impact 
DOS due to incorrect params passing, this is definately a permanent DOS.

High is suitable for this report due to breaking of core functionality of that contract 

Mitigation 
Requires complex fix which needs to be discussed.