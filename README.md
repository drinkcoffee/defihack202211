# Introduction
An organisation with a small token on Binance Smart Chain (BSC) (also known as BNB Chain)
was hacked. No huge amount at stake, but still meaningful enough for a small startup, 
and definitely reputation killer when fund raising. 

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

Attack Transaction:

* The attacker used a transaction [https://bscscan.com/tx/0x085beaf22438287312d56620973b9c00a82b99a44a6cf1f00ef6c88ab3656464](https://bscscan.com/tx/0x085beaf22438287312d56620973b9c00a82b99a44a6cf1f00ef6c88ab3656464) to
transfer all the LPR token from the safe, then emptied our LP pool on pancake swap. 
* The attacker contract called the following function multicall((address,bytes)[]) (function selector 0xcaa5c23f), 
  executing a series of transactions, starting with withdrawing all the treasury tokens. 

## Analysis
### Attack Contract
Attacker (contract assembler code)[attacker-contract.txt]. More analysis can be done using https://ethervm.io/decompile

(Function selectors)[attack-contract-function-selectors].

### Attack Transaction


