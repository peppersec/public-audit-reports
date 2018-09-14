# Robonomic's smartcontracts audit
## Introduction
### Source code
| Object | Location |
| ------ | -------- |
|Robonomics_contracts | [#cc35a91de187072214d215262d8371f0159c2498](https://github.com/airalab/robonomics_contracts/tree/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics)|


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
    <td><a href=#2-Token-stealing>Token stealing</a></td>
    <td rowspan="1">Critical</td>
    <td>Lighthouse</td>
  </tr>
  <tr>
  <td><a href=#1-Possible-keccak256-collisions>Possible keccak256 collisions</a></td>
    <td>Major</td>
    <td>RobotLiability</td>
  </tr>
  <tr>
  <td><a href=#1-Gas-improvement>Gas improvement</a></td>
    <td rowspan="2">Medium</td>
    <td>Ambix</td>
  </tr>
  <tr>
  <td><a href=#1-Dangerous-function>Dangerous function</a></td>
    <td>Congress (MultiSig)</td>
  </tr>
  <tr>
  <td><a href=#2-Validation-after-storing>Validation after storing</a></td>
  <td rowspan="7">Minor</td>
  <td>Ambix</td>
  </tr>
  <tr>
  <td><a href=#1-Possible-reentracy-at-withdraw>Possible reentracy at withdraw</a></td>
  <td rowspan="2">Lighthouse</td>
  </tr>
  <tr>
  <td><a href=#5-Using-txorigin>Using tx.origin</a></td>
  </tr>
  <tr>
  <td><a href=#2-Possible-reentracy-at-finalize>Possible reentracy at finalize</a></td>
  <td>RobotLiability</td>
  </tr>
  <tr>
  <td><a href=#1-Possible-integer-overflow>Possible integer overflow</a></td>
  <td>LiabilityFactory</td>
  </tr>
  <tr>
  <td><a href=#1-Using-old-Openzeppelin-lib-version>Using old Openzeppelin lib version</a></td>
  <td>XRT</td>
  </tr>
  <tr>
  <td><a href=#2-Solidity-version-too-old>Solidity version too old</a></td>
  <td>Congress (MultiSig)</td>
  </tr>
  <tr>
  <td><a href=#3-Address-colliding-around-zero-index-in-indexOf>Address colliding around zero index in indexOf</a></td>
  <td rowspan="7">Note</td>
  <td rowspan="2">Lighthouse</td>
  </tr>
  <tr>
  <td><a href=#3-Address-colliding-around-zero-index-in-indexOf>Possible integer overflow</a></td>
  </tr>
</table>

## General Findings
### Ambix
#### 1. Gas improvement

Severity: **Medium**

[Ambix.sol#L35](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/Ambix.sol#L35) `appendSource` function can be more Gas effective. It uses the [`for` loop](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/Ambix.sol#L44-L49) to validate input and add elements to arrays. Separation of that actions can give sufficient Gas saving. 

*Recomendations:* `appendSource` may looks like following.
```javascript=
function appendSource(address[] _a, uint256[] _n) public onlyOwner {
    require(_a.length == _n.length);

    for (uint256 i = 0; i < _a.length; ++i) {
        require(_a[i] != 0);
    }
    
    A.push(_a);
    N.push(_n);
}
```
This aproach takes 131191 Gas less per 10 elements than original function. (or $1.26 with GasPrice 20 Gwei)

*Status:* Fixed [#818a90313e32a74dbdd32164281c5a733d49fe76](https://github.com/airalab/robonomics_contracts/commit/818a90313e32a74dbdd32164281c5a733d49fe76)

#### 2. Validation after storing

Severity: **Minor**

[Ambix.sol#L82](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/Ambix.sol#L82) `run` function makes decision accordingly to first item of array `N` (token value coefficients), if it's zero then function starts `Dynamic conversion` and it's supposed  that `A[ix]` and `B[ix]` have length == 1. But there are no guarantees that length is really equal 1, so execution would be halted at [line 109](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/Ambix.sol#L109) in negative case.

At this type of flow, it's still possible to submit values than will never be processed by `run` (length of `A[ix]` and `B[ix]` more than 1 and `N[_ix][0] == 0`).

*Recommendations:* consider validating `A`,`B` and `N` before calling `run` ( within `appendSource` func strictly speaking).

*Status:* Fixed [#6173a0270b844376366a5a54f76134891a6e8a53](https://github.com/airalab/robonomics_contracts/commit/6173a0270b844376366a5a54f76134891a6e8a53)

### Lighthouse
#### 1. Possible reentracy at `withdraw` 

Severity: **Minor**

[LighthouseLib.sol#L24](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L24) `withdraw` function send tokens before changing internal `balance`. This behavior can be exploited to steal tokens if actual Token meets some circumstances - it should have `onTokenTransfer` method (or similar) which calls fallback function of _token reciever_ when actual `transfer` has happened (e.g ERC223 and ERC667). 

*Recommendations:* consider swapping [23 and 24 lines](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L23-L24) and also [28 and 29](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L28-L29) to prevent accidents in future.  

*Status:* Fixed [#5fdd39e7cc2c189fd44bc35bdd977c6ae4577096](https://github.com/airalab/robonomics_contracts/commit/5fdd39e7cc2c189fd44bc35bdd977c6ae4577096)

#### 2. Token stealing 

Severity: **Critical**

[LighthouseLib.sol#L83](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L83) `to` function can be used to make arbitrary call on behalf of `Lighthouse` contract. So, this function can be used for token stealing after someone `approved` some amount of tokens to become `member`. Also `to` can be used for increasing `quota` (by calling `refill` and `to`).

*Recommendations:* consider implementing wrappers for external calls to certain contracts instead of `to` function.

*Status:* Fixed [#eddf51b9948e3c15ab1c70bc74438c608a0d6e6b](https://github.com/airalab/robonomics_contracts/commit/eddf51b9948e3c15ab1c70bc74438c608a0d6e6b)

#### 3. Address colliding around zero index in `indexOf`

Severity: **Note**

[LighthouseLib.sol#L83](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L83) due to using mapping to store member index, for all unknown addresses and first member in `member` array index is `0`. So just be aware of this collision for future development or take countermeasures to prevent accidents.
   

*Recommendations:* consider shifting all indexes to `+1` in `indexOf` mapping.

*Status:* taken into account

#### 4. Possible integer overflow

Severity: **Note**

[LighthouseLib.sol#L52](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L52) `quoted` modifier decrase `quota` variable via `-=`. So, if `nextMember` has _balance_ < `minimalFreeze`, [`quota`](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L48) will be zero and then underflowed (become 2**256). Current `withdraw` does not allow that, but consider using `assert(quota != 0);` before [line 52](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LighthouseLib.sol#L52) to avoid possible accidents (and make my sleep happy).

*Status:* Fixed [#5fdd39e7cc2c189fd44bc35bdd977c6ae4577096](https://github.com/airalab/robonomics_contracts/commit/5fdd39e7cc2c189fd44bc35bdd977c6ae4577096)

#### 5. Using `tx.origin`
Severity: **Minor**

As soon as `Lighthouse` contract controls `LiabilityFactory`, it has the special fallback function to proxy all calls to it. That kind of interactions leads to using `tx.origin` to determine actual caller by `LiabilityFactory` contract. But the using `tx.origin` is [considered as dangerous](https://consensys.github.io/smart-contract-best-practices/recommendations/#avoid-using-txorigin) and not recommended.  

*Recomendations:* consider passing actual caller to call to `LiabilityFactory` (as additional parameter) or check that 
```javascript=
require(msg.sender == tx.origin);
``` 
in [fallback function](https://github.com/airalab/robonomics_contracts/blob/master/contracts/robonomics/LighthouseLib.sol#L86) at least.

*Team comment:* tx.origin is used for bounty transfer only

*Status:* team decided to keep it as is. 

### RobotLiability
#### 1. Possible keccak256 collisions 
Severity: **Major** (can be marked as critical as soon as impact will be approved)

[RobotLiabilityLib.sol#L73](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/RobotLiabilityLib.sol#L73). Function `bid` checks `model` and `objective` that were sent this way:
```javascript=
require(keccak256(abi.encodePacked(model, objective)) == 
        keccak256(abi.encodePacked(_model, _objective)));
```
this approach susceptible to collisions, so it can no be considered as reliable.

Example:
```javascript=
keccak256(abi.encodePacked("\x60\x8b","\x00\x29")) ==
keccak256(abi.encodePacked("\x60","\x8b\x00\x29")) // true
```

*Recommendations:* use `abi.encode` instead of `abi.encodePacked`, so that info about `length` would be included into hash.

*Status:* Fixed [#eddf51b9948e3c15ab1c70bc74438c608a0d6e6b](https://github.com/airalab/robonomics_contracts/commit/eddf51b9948e3c15ab1c70bc74438c608a0d6e6b)

#### 2. Possible reentracy at `finalize` 

Severity: **Minor**

[RobotLiabilityLib.sol#L102](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/RobotLiabilityLib.sol#L102) `finalize` function send tokens before changing `isFinalized` to `true` (see clarifications [above](#1-Possible-reentracy-at-withdraw)) 

*Recommendations:* consider moving [line 135](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/RobotLiabilityLib.sol#L135) to line 112.  

*Status:* Fixed [#5fdd39e7cc2c189fd44bc35bdd977c6ae4577096](https://github.com/airalab/robonomics_contracts/commit/5fdd39e7cc2c189fd44bc35bdd977c6ae4577096)

### LiabilityFactory 
#### 1. Possible integer overflow 

Severity: **Minor**

[LiabilityFactory.sol#L222](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/contracts/robonomics/LiabilityFactory.sol#L222) `liabilityFinalized` function does not check input arg. In case of `_gas` less than `gasleft()` LiabilityFactory mints huge amount of tokens (because of integer overflow).

*Recomendations:* due to current workflow there is no suitable way to exploit that function. But consider adding `assert(_gas >= gasLeft())` to avoid future accidents.   

*Status:* Fixed [#1e30dfe6182b46e03e70d05a44c986ca9d47bd88](https://github.com/airalab/robonomics_contracts/commit/1e30dfe6182b46e03e70d05a44c986ca9d47bd88)

### XRT
#### 1. Using old Openzeppelin lib version 
Severity: **Minor**

The [project](https://github.com/airalab/robonomics_contracts/blob/cc35a91de187072214d215262d8371f0159c2498/package.json#L27) uses old version OpenZeppelin lib.

*Recommendations:* consider updating lib to get current improvements and patches.

*Status:* Fixed [23b4227a8eab214f2abb1b19913d5f295c25c71c](https://github.com/airalab/robonomics_contracts/commit/23b4227a8eab214f2abb1b19913d5f295c25c71c)

### Congress (MultiSig)
#### 1. Dangerous function

Severity: **Medium**

[Congress.sol#L133](https://etherscan.io/address/0x97282A7a15f9bEaDC854E8793AAe43B089F14b4e#code). Function `receiveApproval` is used for receive tokens that have been approved by someone. That function can be called by anyone, with an arbitrary Token address. And then contract just call `transferFrom` func of it. That approach is dangerous because there is no guarantee that _Token address_ is an address of a real Token. In case of control over certain smart contract by Congress multisig, an attacker could use `receiveApproval` func to call that contract on behalf of Congress multisig, which may lead to unexpected consequences.

*Recommendations:* consider using `executeProposal` for that purpose or adding access control modifier for `receiveApproval` at least.

*Status:* taken into account

#### 2. Solidity version too old

Severity: **Minor**

The version (v0.4.9+commit.364da425) which was used to contract is too old. The modern version has a bunch of improvements that can be useful for the contract:
* keywords `constructor`, `require`, `emit` to keep code more readable
* `abi.encode()` function to encode args. That could be used to prepare args before hashing (lines 368, 393, 444)
* fixed compiler bugs (e.g for [zero string literal](https://etherscan.io/solcbuginfo?a=SkipEmptyStringLiteral) which is used in the contract)

*Recommendations:* consider improving the code in case of redeploying.

*Status:* taken into account

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
