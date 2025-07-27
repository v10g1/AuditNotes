it is cross chain protocol.
Explanation of each function will be given below
Start date -> 26/07/25
End date-> 28/07/25
3 days
nSLOC -> ~550
Thanks
## Deposit function
It allows the user to deposit the token, if that token is whitelisted and according to the ReadME the only token whiteListed right now is `USDT` 
It requires that the Param `Index` should be less than the `NUM_TOKENs` for which in our can is equal to 4.
It also requires that the the fetched pool should not be equal to zero address.

It also does something very interesting, it withdraws the reward from the pool contract and deposits the rest of the balance of this contract back to the pool.

Then deposits all the tokens gotten by the user transfer to the pool.

That will make the difference in the balanceOf the contract, this balance of will be used as the Minting parameter as follow.
Note -> It gives  the difference (VirtualAmountBefore - virtualAmountAfter) to the `_mintAfterTotalChanged` function which further give THE SAME param to the `_FromVirtualAfterTotalChangedForMint` function 
```javascript
function _mintAfterTotalChanged(address account, uint virtualAmount, uint index) internal virtual {
uint realAmount = _fromVirtualAfterTotalChangedForMint(virtualAmount, index);
if (realAmount == 0) {

            return;
}
return MultiToken._mint(account, realAmount, index);
}
```


### Calculation of minting amount
there are few definition in the calculating function
realTotal=> Total supply of a specific sub token.
`virtualAmout` => `virtualAmountAfter`-`virtualAmountBefore`
`totalvirtualamount` 
`totalVirtual` = `totalVirutalAmount` -`virtualamount`
`totalVirtual` => `totalVirtualAmount` -  (`virtualAmountAter` -`virtualAmountBefore`)
Now `totalVirtualAmount` is the current amount of the balance of the contract which is equal to the `vitualAmountAfter`
`totalVirtual` - `totalVirtualAmount` + `virtualAmountBefore` = `virtualAmountBefore` 

the final function which we can see is the assemble block comes out to be 
```javascript
assembly {
out := div(mul(virtualAmount, realTotal), totalVirtual)
}
```
the formula comes out to be 
(`virtualAmountAfter`-`virtualAmountBefore`) X (total Supply of minting token)/`virtualAmountBefore`

What if its the first mint?
Will Mint VirtualAmount

Can there be inflation attack?
Yes, its possible because of the reason that anyone can directly send some amount of USDT, to the contract and then call the deposit function with minimal amount, this way the LP will be given as 1wei + 1000 Token USDT nd as long as the first submission is less than 1000 usdt the first submitter will get 0 shares while the attacker will have one shares which will result in ALL The funds be attacked buy the attacker

## Withdraw Function
Anyone can call the withdraw function for their OWN withdrawal.
When withdrawing the tokens, all the tokens withdrawn are in the same ratio, paassing the amount would trigger the function for all token which will withdraw individual tokens
from the respective pools
`_withdrawIdex` function is used for anything related to withdrawal of tokens

the formula used for the calculation of `subVirtualAmount`

`subVirtualAmount`=$\frac{\text{totalVirtual} \times \text{RealAmount}}{\text{realTotal}}$
which is further used

`SubVirtualAmount`=$\frac{\text{virtualAmount} \times \text{subVirtualBalance}}{\text{totalVirtualBalance}}$

Now the main function which actuaally tranfers the function is
`_subWithdraw`
Burns the respective tokens (For indexes), 
**Questions**
1. Why are we burning tokens before ?
Answer: We are burning tokens optimistically but there is no protection against burning unintensional token.
2. Is there  a way the user receive lesser token then intended?
Answer: Yes there is a way if we can recieve lesser amount of tokens from the pool then intended
3. Does it consider the case where the tokens recieved is less but the burned tokens are not 
No

I don't think the above scenario is possible because of the way that the virtualAmount is calculated

## Overview till now

Till now from what i can understand the, protocol tries to do these things
- Wrap the around the main Pool.
- It gives Hypothetic token for the submitting an X amount of index (0-3) tokens
- deposits and withdrawal can be done at any point of time no matter the state of the pool, it does not consider the case that the pool is paused