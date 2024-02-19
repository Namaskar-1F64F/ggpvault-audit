# Always Be Growing

ABG is a talented group of engineers focused on growing the web3 ecosystem. Learn more at https://abg.garden

# Introduction

The following is a security review of the GGP Vault contract.

The GGP Vault contract is an [ERC4626](https://eips.ethereum.org/EIPS/eip-4626) compliant vault with the [GoGoPool protocol](https://www.gogopool.com/) token, [GGP](https://docs.gogopool.com/design/how-minipools-work/ggp-rewards), as an underlying asset. The funds are utilized in a strategy staking GGP tokens on behalf of trusted node operators and receiving the rewards generated as yield.

Upon deposit of GGP, a number of receipt tokens of xGGP will be mint. These tokens represent the share of assets held within the contract (referred to henceforth as the "vault"). The vault will use the GGP tokens to generate yield. The strategy for generating yield involves staking the GGP tokens on behalf of trusted node operators participating in the GoGoPool protocol. The GoGoPool protocol provides rewards for staking and those will be accrued by the vault, and increase the value of the xGGP tokens. The xGGP tokens can be redeemed for the underlying GGP tokens.

_Disclaimer:_ This security review does not guarantee against a hack. It is a snapshot in time of brink according to the specific commit by a single human. Any modifications to the code will require a new security review.

# Architecture

The vault inherits much of its base functionality from trusted and audited contracts from the [OpenZeppelin contract library](https://www.openzeppelin.com/contracts). However, the vault has been modified to incorporate features for added security and ability to interact with the GoGoPool protocol. The vault has been designed to be upgradable, allowing for future improvements and bug fixes.

The areas of focus therefore are:

1. Does the contract adhere to the ERC-4626 standard?
2. Do the security features work as intended?
3. Are the changes to the OpenZeppelin contracts secure?

# Methodology

The following is steps I took to evaluate the change.

- Clone & Setup project
- Read Readme and related documentation
- Line by line review
  - Contract
  - Interface
  - Unit Tests
  - Integration Tests
- Review Deployment Strategy
- Review Operation Strategy
- Run Slither
- Run ERC-4626 Test Suite

# Findings

## Critical Risk

No findings

## High Risk

No findings

## Medium Risk

No findings

## Low Risk

2 findings

### Unprotected upgradeable contract

**Severity:** Medium

**Context:** [`GGPVault.sol`](https://github.com/SeaFi-Labs/GGP-Vault/commit/93b1f825014c08117194204a292ec440dfd52f1b#diff-9dc028acb86f9a08936b7224720744d400aef5016bf130f158c6882dda407f33)

**Description:**
While the initializer function is protected by the `initializer` modifier, preventing the re-initialization of the contract, the implementation contract has not been initialized. This means that the contract can be initialized by anyone, potentially causing unexpected behavior.

**Recommendation:**
The implementation contract should be initialized in the constructor of the proxy contract. This will prevent the contract from being initialized by anyone other than the deployer.

```solidity
constructor(){
    _disableInitizliers();
}
```

**Resolution:**
The recommended code was added to the contract in commit [9ebc70d](https://github.com/SeaFi-Labs/GGP-Vault/commit/9ebc70dcc9716e10bdfeceb0d102fc13a7eae526#diff-c25a2ac5ee4834d11f52576c9dff4f8a8f85ec3b890b020cd43b13ed8f3f720dR50).

### Lack of input validation

**Severity:** Low

**Context:** [`GGPVault.sol#L81`](https://github.com/SeaFi-Labs/GGP-Vault/commit/93b1f825014c08117194204a292ec440dfd52f1b?diff=unified&w=0#diff-9dc028acb86f9a08936b7224720744d400aef5016bf130f158c6882dda407f33R81)

**Description:**
The `setAssetCap` (and `setTargetAPR` in a future version) functions do not validate input beyond the access control checks. While not directly a vulnerability, improper setting of these values could disrupt the contract's economics or functionality.

**Recommendation:**
Add input validation to the `setGGPCap` and `setTargetAPR` functions to ensure that the inputs are within the expected range.

## Informational

1 finding

### Shadowing Variables

**Severity:** Informational

**Context:** [`GGPVault.sol#L140`](https://github.com/SeaFi-Labs/GGP-Vault/commit/93b1f825014c08117194204a292ec440dfd52f1b?diff=unified&w=0#diff-9dc028acb86f9a08936b7224720744d400aef5016bf130f158c6882dda407f33R140)

**Description:**

The vault inherits from the `OwnableUpgradeable` contract, which has a function `owner`. It is good practice to avoid using the same name for newly declared variables, as it can lead to confusion and unexpected behavior. This is not a security risk for the vault with this particular code, but it can be confusing to developers and may lead to unexpected behavior.

```solidity
function maxRedeem(address owner) public view override returns (uint256) {
    // `owner` here shadows the `owner` function from OwnableUpgradeable
}
```

**Recommendation:**
Rename the variable to something that does not shadow the `owner` variable from `OwnableUpgradeable`. I recommend `shareHolder` as it is more descriptive of the variable's purpose.

```diff
- function maxRedeem(address owner) public view override returns (uint256)
+ function maxRedeem(address shareHolder) public view override returns (uint256)
```

**Resolution**
The instances of `owner` variable have been changed to `shareHolder` in commit [cc704a5](https://github.com/SeaFi-Labs/GGP-Vault/commit/cc704a55599895398dbd16818cacdc9c08f29176#diff-9dc028acb86f9a08936b7224720744d400aef5016bf130f158c6882dda407f33R136-R149)

# Additional Risks

There are some additional risks. While they are not directly related to the code, they are still important to consider.

## GGP Price fluctuations

Given the volitile price of the GGP token, the valut will ultimately be exposed to those price changes. The direct consequence of these changes can be seen in the node's collateralization raio. If the price drops significantly, the node will be under collateralized and will receive the required rewards, however the GGP Cap will be artifically low, and overall have reward inefficiency. On the other hand, if the price increases significantly, the node will be over collateralized, the GGP Cap will be too high and will accrue more in rewards than in actuality. This delta could potentially lead to insufficient liquidity in the vault.

## Upgradable Contracts

### GGP Vault

The vault itself if upgradable. This introduces a risk of the contract changing at any time by the owner. Additionally, there is added complexity which could lead to a faulty upgrade. This process can be mitigated by having a robust upgrade process, including a well-tested contract before the upgrade.

### GoGoPool Contracts

This contract interacts with the GoGoPool protocol. The GoGoPool protocol is also upgradable. This introduces a risk of the contract changing at any time by the owner. Due to this, the contracts that are being interacted with: the storage and the staking contracts, should be monitored by the team for changes.

## External Contract Interaction

### GGP Token

During normal operations of the vault, the transfers and approvals to the GGP token are done in a safe manner. The GGP token has been [audited](https://docs.gogopool.com/security/audits) and follows the ERC20 standard.

### Staking Contract

The vault interacts with the GoGoPool protocol. In `GGPVault.stakeAndDistributeRewards`, the external function call `stakingContract.stakeGGPOnBehalfOf`, calls the GoGoPool protocol. This is a trusted contract without a risk of re-entry. However, as mentioned, the GoGoPool protocol is upgradable. This introduces a risk of the contract changing at any time by the owner. Due to this, the contracts that are being interacted with: the storage and the staking contracts, should be monitored by the team for changes.

## Centralization Risk

### Elevated Privileges

The owner of the contract as well as the trusted node operators have a large amount of control over the vault. The owner can upgrade the contract, and the trusted node operators can change the configuration of the vault. This introduces a risk of centralization. The team should consider ways to mitigate this risk for the long term health of the vault.

### Role Setting

Given the power of the elevated roles, the setting of new node operators or changing the owner must be done with caution.

## Other Misc

The Solidity version ^0.8.20 used in the vault is potentially too new to be fully trusted.

Test coverage is complete.

Code is well documented with NatSpec and the documentation site.

## Conclusion

The only findings were of low and informational severity. The low severity findings were a potential griefing in the implementation contract and un-bound variable setting. The informational severity finding was a shadowed variable. The contract has been reviewed and tested thoroughly, and the findings have been addressed. There are additional risks to consider, such as the GGP price fluctuations, the upgradability of the contracts, and the centralization risk.

I believe the answer to the questions I set out to answer are yes. The contract adheres to the ERC-4626 standard, the security features work as intended, and the changes to the OpenZeppelin contracts are secure.
