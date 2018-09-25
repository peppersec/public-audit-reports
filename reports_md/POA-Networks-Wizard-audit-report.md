# POA Network Token wizard's smart contracts audit
## Introduction
### Source code
| Object | Location |
| ------ | -------- |
|Token Wizard App | [#2840b97dea33c8cf455a67b2b9c7229e2cda1843](https://github.com/poanetwork/auth-os-applications/tree/2840b97dea33c8cf455a67b2b9c7229e2cda1843)|
|Auth_os | [release #1.0.4](https://github.com/auth-os/core/tree/cebb1089c417a8e26bd97a44f7234bdb9d0bd781) |

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
    <td><a href=#1-lack-of-access-control>Lack of access control</a></td>
    <td rowspan="1">Critical</td>
    <td>Auth-os: ScriptExec</td>
  </tr>
  <tr>
    <td><a href=#1-contract-does-not-prevent-accidental-ether-transfer>Contract does not prevent accidental Ether transfer</a></td>
    <td rowspan="1">Major</td>
    <td>DutchProxy</td>
  </tr>
  <tr>
    <td><a href=#1-math-improvement>Math improvement</a></td>
    <td rowspan="13">Minor</td>
    <td rowspan="3">Auth-os: Contract</td>
  </tr>
  <tr>
    <td><a href=#2-code-reusing>Code reusing</a></td>
  </tr>
  <tr>
    <td><a href=#3-redundant-code>Redundant code</a></td>
  </tr>
  <tr>
    <td><a href=#3-code-reusing>Code reusing</a></td>
    <td>Auth-os: Abstract storage</td>
  </tr>
  <tr>
    <td><a href=#1-documentation-mistype-or-logical-flaw>Documentation mistype or logical flaw</a></td>
    <td>DutchCrowdsale: Token</td>
  </tr>
  <tr>
    <td><a href=#1-there-is-no-iswhitelisted-check-during-purchase>There is no “isWhitelisted” check during purchase</a></td>
    <td>DutchCrowdsale: Sale</td>
  </tr>
  <tr>
    <td><a href=#1-documentation-mistype-1>Documentation mistype</a></td>
    <td rowspan="3">DutchCrowdsale: Admin</td>
  </tr>
  <tr>
    <td><a href=#2-there-is-no-check-that-_min_token_purchase--_max_token_purchase>There is no check that _min_token_purchase <= _max_token_purchase</a></td>
  </tr>
  <tr>
    <td><a href=#3-code-reusing-1>Code reusing</a></td>
  </tr>
  <tr>
    <td><a href=#1-code-reusing>Code reusing</a></td>
    <td rowspan="2">DutchCrowdsaleIdx</td>
  </tr>
  <tr>
    <td><a href=#2-there-are-no-overflow-checks>There are no overflow checks</a></td>
  </tr>
  <tr>
    <td><a href=#1-whitelist-can-be-added-to-a-non-existent-tier>Whitelist can be added to a non-existent tier</a></td>
    <td>MintedCappedCrowdsale: SaleManager</td>
  </tr>
  <tr>
    <td><a href=#1-unnecessary-functionality>Unnecessary functionality</a></td>
    <td>ProxiesRegistry</td>
  </tr>
  <tr>
    <td><a href=#1-documentation-mistype>Documentation mistype</a></td>
    <td rowspan="6">Note</td>
    <td rowspan="3">Auth-os: Abstract storage</td>
  </tr>
  <tr>
    <td><a href=#2-payment-can-be-delivered-via-transfer-only>Payment can be delivered via transfer only</a></td>
  </tr>
  <tr>
    <td><a href=#4-contractsender-can-be-spoofed-exec-func>Contract.Sender() can be spoofed (exec func)</a></td>
  </tr>
  <tr>
    <td><a href=#2-lack-of-input-validation>Lack of input validation</a></td>
    <td>Auth-os: ScriptExec</td>
  </tr>
  <tr>
    <td><a href=#1-token-wizard-app-does-not-use-authos-killer-feature>Token wizard app does not use authos killer feature</a></td>
    <td>General issues</td>
  </tr>
</table>

## General Findings
### Auth-os: Contract
#### 1. Math improvement

Severity: **Minor**

[Contract.sol#L494](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L494) Consider using `>=` sign instead of `>` 

*Status:* Fixed [#da5361fdc0d962e4094e62f78259578d6a15a6ef](https://github.com/auth-os/core/commit/da5361fdc0d962e4094e62f78259578d6a15a6ef)

#### 2. Code reusing

Severity: **Minor**

1. Snippet of code bellow is used 16 times within [Contract.sol](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol) contract: 
    ```javascript
    // If the free-memory pointer does not point beyond the     buffer's current size, update it
    if lt(mload(0x40), add(0x20, add(ptr, mload(ptr)))) {
        mstore(0x40, add(0x20, add(ptr, mload(ptr))))
    }
    ```
    Consider take it into separate func.
    ```javascript
    function setmptr() internal pure {
        assembly {
            let ptr := add(0x20, mload(0xc0))
            if lt(mload(0x40), add(0x20, add(ptr, mload(ptr)))) {
                mstore(0x40, add(0x20, add(ptr, mload(ptr))))
            }
        }
    }
    ```
    And call that after `assembly` block or like parameter for `condition` modifier which actually it is.

2. Consider calling [`initialize()`](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L76) instead of this [code block](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L31-L48) at `authorize` func. 


*Status:* Fixed [#da5361fdc0d962e4094e62f78259578d6a15a6ef](https://github.com/auth-os/core/commit/da5361fdc0d962e4094e62f78259578d6a15a6ef)

#### 3. Redundant code

Severity: **Minor**

The line [Contract.sol#L333](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L333) is duplicate for [Contract.sol#L327](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L327). Since the framework is upon heavy development - code can be replicated many times in the future, which is not a good thing. 

The same thing can be found [here](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L533) and [here](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Contract.sol#L749).

*Recommendations:* just remove it :)

*Status:* Fixed [#de8aabdc8e8d6c81e7b1b2d814856488a7cd9057](https://github.com/auth-os/core/commit/de8aabdc8e8d6c81e7b1b2d814856488a7cd9057)

### Auth-os: Abstract storage
#### 1. Documentation mistype

Severity: **Note**

[AbstractStorage.sol#L452](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/AbstractStorage.sol#L452) This parameter supposed to be `n_emitted` instead of `n_paid`.
[AbstractStorage.sol#L414](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/AbstractStorage.sol#L414) the same thing.
[AbstractStorage.sol#L64](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/AbstractStorage.sol#L64) missed `_provider` description

*Status:* Fixed [#fca016e0aa01f177d3d2d28070d1c80ca43091eb](https://github.com/auth-os/core/commit/fca016e0aa01f177d3d2d28070d1c80ca43091eb)

#### 2. Payment can be delivered via transfer only

Severity: **Note**

[AbstractStorage.sol#L392](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/AbstractStorage.sol#L392) `doPay` func implements payments via `transfer` only. Consider adding `send` functionality. It can be extremely useful for some contracts.

*Team comment:* currently we don't have a way for users to decide what happens if a send fails, so I really think it needs to be either _success_ or throw.

*Status:* Due to current AuthOS architecture, it takes too much to implement `send` behavior, so all developers just should take it into account.

#### 3. Code reusing

Severity: **Minor**

[AbstractStorage.sol#L574](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/AbstractStorage.sol#L574) Consider using this:
```javascript
  function readMulti(bytes32 _exec_id, bytes32[] _locations) public view returns (bytes32[] data_read) {
    data_read = new bytes32[](_locations.length);
    for (uint i = 0; i < _locations.length; i++) {
      data_read[i] = read(_locations[i], _exec_id); // call of `read`
    }
  }
```
instead of actual one. Since the framework is upon heavy development - code reusing is a good approach to minimize amount of bugs.

*Status:* Fixed [#fca016e0aa01f177d3d2d28070d1c80ca43091eb](https://github.com/auth-os/core/commit/fca016e0aa01f177d3d2d28070d1c80ca43091eb)

#### 4. `Contract.Sender()` can be spoofed (`exec` func)

Severity: **Note**

Due to architectural feature, any function can be called via `AbstactStorage.exec(address sender,...)` instead of `RegistryExec.exec` or `DutchProxy`. For example, the `Token.trasfer` function [uses](https://github.com/poanetwork/auth-os-applications/blob/308a6a43d187d84e54de3da0d3c714a20b4fa329/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/token/Token.sol#L36) `Contract.sender()` to get caller (considered to `_from`), but it just an arbitary value that caller can send (see `AbstactStorage.exec` above). Actual authorization is implemented by `Token` contract itself - it checks that actual caller is `Proxy` contract. 

Same story about `createInstance` func.

*Recommendation:* All developers that use AuthOS should be aware of this behavior. Consider adding this into documentation.

*Team comment:* This is intended, and is in fact used in the latest RegistryExec. Maybe a better name for the variable would be "exec_as" or something. Basically, it is part of the architecture to use a ScriptExec contract, or something similar to interface with storage, and it is up to that contract to perform input validation and provide information to storage. It's just a separation of concerns problem, and the job here is given to ScriptExec.

*Status:* There is no issue here. Just the thing which should be taken into account. 

### Auth-os: ScriptExec
#### 1. Lack of access control

Severity: **Critical**

There is no `onlyOwner` modifier at [configure](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/ScriptExec.sol#L55) function. So an attacker can use it to reconfigure app.

*Recommendation:* add some access control for the func.

*Status:* Fixed [#d11890df8628682099af4ebc4743c8db948252bf](https://github.com/auth-os/core/commit/d11890df8628682099af4ebc4743c8db948252bf).

#### 2. Lack of input validation

Severity: **Note**

In contrast to other functions and [parameters](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/ScriptExec.sol#L56), [`configure`](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/ScriptExec.sol#L55) and [`setProvider`](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/ScriptExec.sol#L188) funcs do not check  `provider` address.

*Recommendation:* consider adding some checks to keep code uniform.

*Team comment:* `_provider` check is unnecessary, as any address might be a valid provider, even 0x0.

*Status:* there is no issue here.

### DutchCrowdsale: Token
#### 1. Documentation mistype or logical flaw

Severity: **Minor**

The line [Token.sol#L198](https://github.com/poanetwork/auth-os-applications/blob/308a6a43d187d84e54de3da0d3c714a20b4fa329/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/token/Token.sol#L198) tell us "Ensures state change will ***only*** affect storage and events". But actual [`emitAndStore`](https://github.com/poanetwork/auth-os-applications/blob/308a6a43d187d84e54de3da0d3c714a20b4fa329/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/token/Token.sol#L178) func just checks that `emitted` and `stored` buffers are not empty (note, there is no `payment` check here). So, if a function does some unexpected manipulations with `payment` buffer it won't be noticed here (but comment tell us the opposite thing).

*Status:* Fixed  [#9e21ef2ddc536ab9701e67db229caa8d02c2e5de](https://github.com/poanetwork/auth-os-applications/commit/9e21ef2ddc536ab9701e67db229caa8d02c2e5de)

### DutchCrowdsale: Sale
#### 1. There is no "isWhitelisted" check during purchase
Severity: **Minor**

[Sale.sol#L292](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/sale/Sale.sol#L292) `buy` function does not check a contributor being/not being whitelisted, so `min_contribution` is set [to 0](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/sale/Sale.sol#L34) during execution. Fortunately, that behavior is not exploitable because of zero amount exception at [line 144](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/sale/Sale.sol#L144).

*Recommendation:* Consider adding an explicit check that emits readable exception message.

*Status:* Team decided to leave it as-is. There is no danger. 

### DutchCrowdsale: Admin
#### 1. Documentation mistype
Severity: **Minor**

[Admin.sol#L378](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/admin/Admin.sol#L378) `_max_wei_spend` parameter is used as [`_max_token_purchase`](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/admin/Admin.sol#L63) actualy. 

*Recommendation:* Rename the parameter to avoid misunderstanding.

*Status:* Fixed [#9e21ef2ddc536ab9701e67db229caa8d02c2e5de](https://github.com/auth-os/applications/commit/9e21ef2ddc536ab9701e67db229caa8d02c2e5de)

#### 2. There is no check that `_min_token_purchase` <= `_max_token_purchase`
Severity: **Minor**

[Admin.sol#L63](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/admin/Admin.sol#L63) `whitelistMulti` func does not check that `_min_token_purchase` <= `_max_token_purchase`, so they can keep any values. At this time there is no serious impact here. 

*Recommendation:* consider adding that check to avoid accidents.

*Status:* Fixed [#9e21ef2ddc536ab9701e67db229caa8d02c2e5de](https://github.com/auth-os/applications/commit/9e21ef2ddc536ab9701e67db229caa8d02c2e5de)

#### 3. Code reusing
Severity: **Minor**

[Admin.sol#L322-L323](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/admin/Admin.sol#L322-L323) consider calling `onlyAdmin` func instead of a copy-pasting function body. Same thing [here](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/classes/admin/Admin.sol#L331-L332).

*Status:* Fixed [#9e21ef2ddc536ab9701e67db229caa8d02c2e5de](https://github.com/auth-os/applications/commit/9e21ef2ddc536ab9701e67db229caa8d02c2e5de)

### DutchCrowdsale: DutchProxy
#### 1. Contract does not prevent accidental Ether transfer

Severity: **Major**

`DutchProxy` contract has the `payble` fallback function (inherits the [Proxy.sol#L26](https://github.com/auth-os/core/blob/cebb1089c417a8e26bd97a44f7234bdb9d0bd781/contracts/core/Proxy.sol#L26)) _for storage refunds_. But this is not a good idea for crowdsale app - due to user experience, someone can just send Ether to "crowdsale" address and lose it(`buy` func will not be called).

*Recommedation:* Consider using another (custom) func to refund or fallback func should check that `msg.sender == address(app_storage)` 

*Status:* Fixed [#fd207315418bdc4b16482516ae6d4e6df7b0a801](https://github.com/auth-os/core/commit/fd207315418bdc4b16482516ae6d4e6df7b0a801)

### DutchCrowdsale: DutchCrowdsaleIdx
#### 1. Code reusing

Severity: **Minor**

Consider importing `Token`, `Sale` and `Admin` libraries instead of [copy-pasting](https://github.com/poanetwork/auth-os-applications/blob/308a6a43d187d84e54de3da0d3c714a20b4fa329/TokenWizard/crowdsale/DutchCrowdsale/contracts/DutchCrowdsaleIdx.sol#L19-L140) their functionality. After importing, it can be used the same way as `Contract` (e.g. `Contract.storing()`). So it would be 
```javascript=
Contract.set(Sale.startRate()).to(_starting_rate);
```
instead of 
```javascript=
Contract.set(startRate()).to(_starting_rate);
```
which is more readable at my point of view and keeps code minimalistic.

*Status:* taken into account 

#### 2. There are no overflow checks

Severity: **Minor**

[DutchCrowdsaleIdx.sol#L362](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/DutchCrowdsaleIdx.sol#L362) `getRateAndTimeRemaining` func does not check `_start_rate` and `_end_rate` values, so if `_end_rate` is bigger than `_start_rate` then (at [line 377](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/DutchCrowdsale/contracts/DutchCrowdsaleIdx.sol#L377)) `uint` underflow occurs and current rate becomes huge. 

*Recomendation:* At this moment there is no way to exploit that behavior, but this kind of check (`_start_rate` >= `_end_rate`) would be extremely useful to complicate or eliminate attacks. 

*Status:* Team decided to leave it as-is. There is no danger. 

### MintedCappedCrowdsale: SaleManager
#### 1. whitelist can be added to a non-existent tier
Severity: **Minor**

[SaleManager.sol#L145](https://github.com/poanetwork/auth-os-applications/blob/2840b97dea33c8cf455a67b2b9c7229e2cda1843/TokenWizard/crowdsale/MintedCappedCrowdsale/contracts/classes/sale_manager/SaleManager.sol#L145). `whitelistMultiForTier` func does not check `_tier_index`, so it can be any.

*Recommendation:* consider adding that check to avoid accidents. i.e 
*current_tier_index* <=`_tier_index` <= *last_tier_index*

*Status:* Team decided to leave it as-is. There is no danger. 

### ProxiesRegistry
#### 1. Unnecessary functionality

Severity: **Minor**

The new version of openzeppelin `Ownable` contract has `renounceOwnership` function. See [here](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/5daaf60d11ee2075260d0f3adfb22b1c536db983/contracts/ownership/Ownable.sol#L42). This function is inherited by your `ProxiesRegistry` silently. `renounceOwnership` seems to be superfluous ([not only for me](https://github.com/OpenZeppelin/openzeppelin-solidity/issues/903)). Is that necessary for your project? 

*Recomendations:* Consider rewrite `renounceOwnership` to empty implementation. 

*Status:* Fixed [#f9d1518369d37a34bb1685416977cb891f908b1b](https://github.com/poanetwork/auth-os-applications/commit/f9d1518369d37a34bb1685416977cb891f908b1b)

### General issues
#### 1. Token wizard app does not use authos killer feature
Severity: **Note**

At this moment, Token and Crowdsale together is just one app. But because of architectural feature (AbstactStorage) they can act as separated apps that share Storage. This behavior can bring an additional layer of security.

*Recommendations:* It can take too much to rewrite TokenWizard, so that is up to develop team. 

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
