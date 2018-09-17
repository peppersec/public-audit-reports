# POA Network bridge's smart contracts audit
## Introduction
### Source code


| Object | Location |
| ------ | -------- |
|POA Network bridge. branch `refactor_v1`  | [#2bf70c7e9fd42968aec2dc352017618907834401](https://github.com/poanetwork/poa-bridge-contracts/tree/2bf70c7e9fd42968aec2dc352017618907834401)|

### Audit Methodology
The code of a smart contract is automatically and manually scanned for known vulnerabilities and logic errors that can lead to security threats. The conformity of requirements (e.g., White Paper) and practical implementation is reviewed as well. More about the methodology [here](https://peppersec.com/smart-contract-audit.html).
### Auditors
* Alexey Pertsev. [PepperSec](https://peppersec.com).

## Overview
The table below contains a summary of found bugs/issues.
<table>
  <tr>
    <th>Vulnerability description</th>
    <th>Severity</th>
    <th>Paragraph</th>
  </tr>
  <tr>
    <td>Double spending</td>
    <td rowspan="1">Major</td>
    <td><a href=#1-double-spending>General issues</a></td>
  </tr>
  <tr>
    <td>Contract does not prevent an accidental token transfer</td>
    <td rowspan="2">Medium</td>
    <td><a href=#2-contract-does-not-prevent-an-accidental-token-transfer>ERC677BridgeToken</a></td>
  </tr>
  <tr>
    <td>Denial of service possibility</td>
    <td><a href=#2-denial-of-service-possibility>HomeBridgeNativeToErc</a></td>
  </tr>
  <tr>
    <td>Unnecessary functionality</td>
    <td rowspan="4">Minor</td>
    <td><a href=#1-unnecessary-functionality>ERC677BridgeToken</a></td>
  </tr>
  <tr>
    <td>Redundant checks/code</td>
    <td><a href=#1-redundant-checkscode>BridgeValidators</a></td>
  </tr>
  <tr>
    <td>Function does not generate the event</td>
    <td><a href=#2-function-does-not-generate-the-event>BridgeValidators</a></td>
  </tr>
  <tr>
    <td>Lack of message checking</td>
    <td><a href=#1-lack-of-message-checking>HomeBridgeNativeToErc</a></td>
  </tr>
  <tr>
    <td>Possibility of Validators/RequiredSignatures desync</td>
    <td rowspan="1">Note</td>
    <td><a href=#2-possibility-of-validatorsrequiredsignatures-desync>General issues</a></td>
  </tr>
</table>

## General Findings
### ERC677BridgeToken
#### 1. Unnecessary functionality

Severity: **Minor**

The new version of openzeppelin `Ownable` contract has `renounceOwnership` function. See [here](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/5daaf60d11ee2075260d0f3adfb22b1c536db983/contracts/ownership/Ownable.sol#L42). So this function is inherited by your `ERC677BridgeToken` silently. The function seems to be superfluous ([not only for me](https://github.com/OpenZeppelin/openzeppelin-solidity/issues/903)). Is that necessary for your project? 

*Recommendations:* Consider rewrite `renounceOwnership` to empty implementation (as you've done for [finishMinting](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/ERC677BridgeToken.sol#L55)). 

*Status:* Fixed [PR48](https://github.com/poanetwork/poa-bridge-contracts/pull/48)

#### 2. Contract does not prevent an accidental token transfer
Severity: **Medium**

Method `transfer` does not prevent tokens transfer to `ForeignBridgeNativeToErc`, but in this case `UserRequestForAffirmation(userAddr, value)` won't be fired.  I see `claimTokens` gonna help to reveal accidentally sent tokens, but prevention of that sending seems to be a good idea though.

*Recomendations:* Implement same [`isContract`](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/ERC677BridgeToken.sol#L31) check and `onTokenTransfer` call at `transfer` method. You can use `_to.call(abi.encodeWithSignature("onTokenTransfer(address,uint256,bytes)",  ...))` instead of `_to.onTokenTransfer(...)` to prevent the `revert` if token receiver has not that method. See example [here](https://gist.github.com/pertsev/7b839682954aa2a1d814554c13783e99).

*Status:* Fixed [PR 50](https://github.com/poanetwork/poa-bridge-contracts/pull/50)

    
### HomeBridgeNativeToErc
#### 1. Lack of `message` checking

Severity: **Minor**

[BasicHomeBridge.sol#L43](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/BasicHomeBridge.sol#L43) Сontract does not check `message` entities within `submitSignature` func. 

*Scenario:* If the attacker can spoof `recipient` or `value` by MITM attack between Ethereum node and Validator bridge software at the time of event grabbing, then Validator will sign and send the `message` with spoofed args. Due to lack of `message` checking, the contract will accept that signature silently. Note: MITM is not only one way to spoof those values, but I'm using that because it's possible right now for `bridge-nodejs`. See configuration [here](https://github.com/poanetwork/bridge-nodejs#env-variables)
    
*Recommendations:* 
1. Safe `msg.sender`, `msg.value` and `TxHash` (optional) at time of Ether receiving ([fallback func](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/native_to_erc20/HomeBridgeNativeToErc.sol#L36)) and check it later (`submitSignature` func).
2. Use https only.

*Roman's answer:* In solidity, there is no way to access txhash, so identifier could only be a hash of `msg.sender, value, now`

*Status:* After discussing all pros and cons of message checking, the team decided that `https` is enough.  

#### 2. Denial of service possibility

Severity: **Medium**

\*Bridge\* contacts don't handle exception situations at the time of token/ether transferring. [HomeBridgeNativeToErc.sol#L45](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/native_to_erc20/HomeBridgeNativeToErc.sol#L45) - if recipient is contact which cannot receive ether (`revert()` at fallback func e.g), then `HomeBridgeNativeToErc` will throw exception and the whole TX will be reverted also. In the case of  Bridge Validators software does not ready to handle that, DOS possible.
    
*Recommendations:* Сountermeasures can be implemented at both smartcontract and Bridge software side. 

1. Bridge Software can "revert" the whole process by generating an opposite transfer.
2. The contract may implement transfer via `selfdestruct` of a child contract. See implementation [here](https://gist.github.com/pertsev/25657874806a216857f4e87515c8fbad) 

*Status:* Fixed [PR51](https://github.com/poanetwork/poa-bridge-contracts/pull/51)

### BridgeValidators
#### 1. Redundant checks/code

Severity: **Minor**

[BridgeValidators.sol#L30](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/BridgeValidators.sol#L30). That check is redundant because of [this](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/BridgeValidators.sol#L22) and [that](https://github.com/poanetwork/poa-bridge-contracts/blob/refactor_v1/contracts/upgradeable_contracts/BridgeValidators.sol#L25). 
    
*Recomendations*: Change `require(...);` to `assert(...);` or remove it.

*Status:* Fixed [PR48](https://github.com/poanetwork/poa-bridge-contracts/pull/48)

#### 2. Function does not generate the event

Severity: **Minor**

`initialize` func set `requiredSignatures` var but does not generate `RequiredSignaturesChanged(requiredSignatures);` event. `setRequiredSignatures` does. Is it just not necessary or missed? `OwnershipTransferred` and `ValidatorAdded` are fired btw.

*Recommendations*: Consider adding the `RequiredSignaturesChanged` emitting.

*Status:* Fixed [PR48](https://github.com/poanetwork/poa-bridge-contracts/pull/48)

### General issues
#### 1. Double spending

Severity: **Major**

Due to lack of difference between signatures for NativeToERC and ERC20ToERC20 (home to foreign) transfers, those can be used for double spending.
    
*Recommendations:* Implement different message for NativeToERC and ERC20ToERC20 transfers. Consider adding `tokenAddress` to  ERC20ToERC20 transfers `message` to avoid double spending and changing token provider contract.

*Status:* Fixed [PR57](https://github.com/poanetwork/poa-bridge-contracts/pull/57)

#### 2. Possibility of Validators/`RequiredSignatures` desync

Severity: **Note**

Since there is no on-chain way to sync the "Home" and "Foreign" sides in terms of "current Validators list" and/or `RequiredSignatures`, Validator Software should notify of any desync and stop Bridge trade if it happens. 

*Team's update:* For this purposes, we use [bridge-monitor](https://github.com/poanetwork/bridge-monitor) to alert if required signatures are not matched.

*Status:* So, there is no issue here.
    
# Appendix 1 - Terminology
## Severity

Assessment of the magnitude of an issue.

|![](https://i.imgur.com/X6N40Ed.png)|
|-----:|
|Picture 1. Severity|


### Minor (Low)

Minor issues are generally subjective in nature or potentially associated with the topics like “best practices” or “readability”. As a rule, minor issues do not indicate an actual problem or bug in the code.

The maintainers should use their own judgment as to whether addressing these issues will improve the codebase.

### Medium

Medium issues are generally objective in nature but do not represent any actual bugs or security problems.

These issues should be addressed unless there is a clear reason not to.

### Major (High)

Major issues are things like bugs or vulnerabilities. These issues may be unexploitable directly or may require a certain condition to arise in order to be exploited.

If unaddressed, these issues are likely to cause problems with the operation of the contract or lead to situations which make the system exploitable.

### Critical

Critical issues are directly exploitable bugs or security vulnerabilities.

If unaddressed, these issues are likely or guaranteed to cause major problems or ultimately a full failure in the operations of the contract.
