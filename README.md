# Introduction
The LPR Token on Binance Smart Chain (BSC) (also known as BNB Chain) was hacked.  
The purpose of this repo is to work through the information available to attempt to 
understand how the hack was executed.

## Presenting Issue
Contracts:
* Gnosis Safe: [0x4b64f382aa063c07f1c55cf53c66cce3b6fd0bb0](https://bscscan.com/address/0x4b64f382aa063c07f1c55cf53c66cce3b6fd0bb0).
  The Gnosis safe has been deployed from the Gnosis UI, no third parties, no modules attached.
  The Gnosis safe was configured with (3/5 signatures).
  Gnosis source code: https://github.com/safe-global/safe-contracts/releases
* The transferred LPR token (that was transfered): [0x91191A15E778d46255FC9AcD37D028228D97e786](https://bscscan.com/address/0x91191a15e778d46255fc9acd37d028228d97e786)
  The token is a basic ERC20, all contracts are from OpenZeppelin library. Just 2 functions were added: batch transfer, and ERC20Permit (ERC20Permit from openZeppelin too).
* The attacker contract: [0xCd62dDE0e5aCbc1D596b1C1699c8b2A5f1327693](https://bscscan.com/address/0xcd62dde0e5acbc1d596b1c1699c8b2a5f1327693)
  Attacker [contract assembler code](attacker-contract.txt). More analysis can be done using 
  [https://ethervm.io/decompile](https://ethervm.io/decompile). Function selectors for the contract
  are contained in the [function selectors file](attack-contract-function-selectors.txt).

Attack Transaction:

* The attacker used a transaction [https://bscscan.com/tx/0x085beaf22438287312d56620973b9c00a82b99a44a6cf1f00ef6c88ab3656464](https://bscscan.com/tx/0x085beaf22438287312d56620973b9c00a82b99a44a6cf1f00ef6c88ab3656464) to
transfer all the LPR token from the safe, then emptied our LP pool on pancake swap. 
* The attacker contract called the following function multicall((address,bytes)[]) (function selector 0xcaa5c23f), 
  executing a series of transactions, starting with withdrawing all the treasury tokens. 

## What happened?
32.412765608378451745 of BNB tokens (approx US$11K) were stolen by draining the LP pool for the 
LPR token on Pancake Swap.

## How did it happen?
There was a bug in the LPR ERC20 contract that allowed the attacker to transfer as many
tokens as they wished to any account. They swapped these stolen LPR tokens for BNB tokens
using Pancake Swap v2.

Analysing the [transaction calldata](attack-transaction.md) revealed that the attacker had 
called ```batchTransferFrom(address,address[],uint256[])``` in the [ERC 20 contract](https://bscscan.com/address/0x91191a15e778d46255fc9acd37d028228d97e786#code). 
The deployed code, as shown below, will transfer from any ```from``` account, without checking
whether the ```msg.sender``` has an allowance to do the transfer. The code could be 
fixed by including the line ```_spendAllowance(from, msg.sender, _value[i]);``` immediately
before the call to _transfer

```
/**
 * @dev Send tokens from one address to multiple recipients in a single tx;
 * @param from Address to send from;
 * @param _to Array of recipient's addresses;
 * @param _value Array of recipient's amounts to receive;
 */
function batchTransferFrom(
        address from,
        address[] memory _to,
        uint256[] memory _value
) public returns (bool) {
  require(_to.length > 0, "No recipient");
  require(_value.length == _to.length, "Array length mismatched");

  for (uint256 i = 0; i < _to.length; i++) {
    _transfer(from, _to[i], _value[i]);
  }
  return true;
}
```

Note: The [transaction trace](https://bscscan.com/vmtrace?txhash=0x085beaf22438287312d56620973b9c00a82b99a44a6cf1f00ef6c88ab3656464&type=gethtrace2) 
was not used, but would have been another way to quickly understand what had happened.

## Who stole the tokens?
This analysis has not been completed. To do this:

* Review all transactions, including deployment transaction, into the attack contract and 
  the other attacker contracts involved in the hack. Check for who msg.sender is and
  what other accounts are interacted with, particularly who the recipients of funds are.
* Check to see if any of the accounts also exist on other chains. 
* Trace back to the funding source of the BNB (BSC's equivalent of Eth), to see how 
  gas has been paid for. 
* When tracing back, you will end up with:
  * An anonymously funded account (via something like Tornado Cash)
  * A bridge. The crosschain transaction will then need to be followed.
  * A fiat to crypto on-ramp. Law enforcement may then be able to engage with the on-ramp 
    company to determine the identities of the account holder. 


## What could be done to avoid this problem?
Some things that could be done:
* Don't add functions to existing, well analysed contracts. ```transfer``` or ```transferFrom```
  could have been called repeatedly by an Externally Owned Account (EOA).
* The ```batchTransferFrom```
  does provide some improvement over the standard ERC 20, allowing one account to 
  distribute tokens to many accounts. The functionality could have been put into a 
  separate contract that had a function that called this action. The function
  could repeatedly called ```transfer``` or ```transferFrom```. Prior 
  to a distribution, the treasury account for the token could transfer enough tokens for 
  the batch transfer. This approach has the advantage that the well reviewed
  ERC 20 contract is not modified.
* A negative test case, where an account that had insufficient approval called the 
  function, would have highlighted the problem.
* More code review may also have highlighted the issue.



