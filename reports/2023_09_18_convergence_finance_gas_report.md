# Convergence Finance Security Review

This security review provides a detailed compilation of all identified vulnerabilities and recommendations submitted as part of the Convergence Finance competition on Hats in September 2023.

|   No.   |   Issue  | Instances |
|---------|----------|-----------|
|[G-01]| Make constructors payable | 3 |
|[G-02]| Admin functions can be payable | 12 |
|[G-03]| Write gas-optimal for-loops | 1 |
|[G-04]| Pre-increment cost less gas than post-increment | 1 |
|[G-05]| Cache storage variables | 1 |
|[G-06]| Constants can be private | 3 |
|[G-07]| Use Solidity 0.8.19 or higher to save gas | 4 |
|[G-08]| Consider choosing a very large value for the optimizer | 1 |
|[G-09]| Split require statements | 1 |

## [G-01] Make constructors payable

### Instances

1. [CvgOracle.sol#L42-L45](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L42-L45)
2. [Ibo.sol#L98-L112](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L98-L112)
3. [VestingCvg.sol#L81-L91](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L81-L91)

### Issue

None of the constructors is payable. 

However, making a constructor payable can save over 200 gas on deployment. This is because non-payable functions have an implicit `require(msg.value == 0)` inserted in them. Additionally, fewer bytecode at deploy time means less gas cost due to smaller calldata.

There are good reasons to make regular functions non-payable, but generally, a contract is deployed by a privileged address that you can reasonably assume won’t send ether. This might not apply if inexperienced users are deploying the contract.

### Recommendation

Assuming that experienced users are deploying the contracts, I'd suggest making the constructors payable as follows:

[CvgOracle.sol#L42](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L42)
```diff
- constructor(address treasuryDao) { 
+ constructor(address treasuryDao) payable {
```

[Ibo.sol#L104](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L104)
```diff
- ) ERC721("IBO Cvg", "cvgIBO") {
+ ) ERC721("IBO Cvg", "cvgIBO") payable {
```

[VestingCvg.sol#L86](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L86)
```diff
- ) {
+ ) payable {
```

Implementing these changes will save a total of 664 gas during deployment.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index a4051d4..612bb4b 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -91,11 +91,11 @@
 ························································|·············|·············|·············|···············|··············
 |  CvgControlTower                                      ·          -  ·          -  ·     859604  ·        2.9 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  CvgOracle                                            ·          -  ·          -  ·    1905972  ·        6.4 %  ·          -  │
+|  CvgOracle                                            ·          -  ·          -  ·    1905748  ·        6.4 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  CvgV3Aggregator                                      ·          -  ·          -  ·     622164  ·        2.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  Ibo                                                  ·          -  ·          -  ·    3102182  ·       10.3 %  ·          -  │
+|  Ibo                                                  ·          -  ·          -  ·    3101968  ·       10.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  mock_CloneFactoryV2                                  ·          -  ·          -  ·     535562  ·        1.8 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     670490  ·     725531  ·     698015  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1852678  ·        6.2 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1852452  ·        6.2 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2977045  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

## [G-02] Admin functions can be payable

### Instances

1. [CvgOracle.sol#L323-L328](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L323-L328)
2. [CvgOracle.sol#L334-L336](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L334-L336)
3. [Ibo.sol#L119-L121](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L119-L121)
4. [Ibo.sol#L128-L131](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L128-L131)
5. [VestingCvg.sol#L115-L117](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L115-L117)
6. [VestingCvg.sol#L119-L121](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L119-L121)
7. [VestingCvg.sol#L123-L125](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L123-L125)
8. [VestingCvg.sol#L127-L129](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L127-L129)
9. [VestingCvg.sol#L131-L133](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L131-L133)
10. [VestingCvg.sol#L171-L214](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L171-L214)
11. [VestingCvg.sol#L220-L227](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L220-L227)
12. [VestingCvg.sol#L402-L407](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L402-L407)

### Issue

None of the admin functions is payable.

However, making admin functions payable can save gas because it skips checking the callvalue.

There are good reasons to make regular functions non-payable, but generally, the admin functions of a contract are only callable by an address that you can reasonably assume won’t send ether. This might not apply if inexperienced users are making these calls.

### Recommendation

As we can see from the Hardhat gas reporter, making this change will optimize the cost of runtime code but increase the deployment costs significantly.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index 40846af..2d98def 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -23,15 +23,15 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  CvgControlTower      ·  setOracle                    ·          -  ·          -  ·      53467  ·            2  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CvgOracle            ·  setCvgToken                  ·          -  ·          -  ·      46095  ·            1  ·          -  │
+|  CvgOracle            ·  setCvgToken                  ·          -  ·          -  ·      46071  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CvgOracle            ·  setTokenOracleParams         ·      71411  ·      71675  ·      71540  ·           31  ·          -  │
+|  CvgOracle            ·  setTokenOracleParams         ·      71387  ·      71651  ·      71516  ·           31  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59525  ·           22  ·          -  │
+|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59526  ·           22  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  ERC20                ·  approve                      ·      26558  ·      46700  ·      44526  ·          119  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  Ibo                  ·  createBond                   ·          -  ·          -  ·      98410  ·            7  ·          -  │
+|  Ibo                  ·  createBond                   ·          -  ·          -  ·      98386  ·            7  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  Ibo                  ·  deposit                      ·     112237  ·     320062  ·     240553  ·           23  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -41,7 +41,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  SeedPresaleCvg       ·  setSaleState                 ·          -  ·          -  ·      28801  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  createVestingSchedule        ·     155747  ·     190007  ·     169458  ·           10  ·          -  │
+|  VestingCvg           ·  createVestingSchedule        ·     155723  ·     189983  ·     169434  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseIbo                   ·      71280  ·     123640  ·      84900  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -51,19 +51,19 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseWl                    ·      75787  ·     128147  ·      88499  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  revokeVestingSchedule        ·          -  ·          -  ·      38463  ·            2  ·          -  │
+|  VestingCvg           ·  revokeVestingSchedule        ·          -  ·          -  ·      38439  ·            2  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setCvg                       ·          -  ·          -  ·      46172  ·            1  ·          -  │
+|  VestingCvg           ·  setCvg                       ·          -  ·          -  ·      46148  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setPresale                   ·          -  ·          -  ·      29094  ·            1  ·          -  │
+|  VestingCvg           ·  setPresale                   ·          -  ·          -  ·      29070  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setPresaleSeed               ·          -  ·          -  ·      29016  ·            1  ·          -  │
+|  VestingCvg           ·  setPresaleSeed               ·          -  ·          -  ·      28992  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setWhitelistDao              ·          -  ·          -  ·      46149  ·            1  ·          -  │
+|  VestingCvg           ·  setWhitelistDao              ·          -  ·          -  ·      46125  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setWhitelistTeam             ·          -  ·          -  ·      46128  ·            1  ·          -  │
+|  VestingCvg           ·  setWhitelistTeam             ·          -  ·          -  ·      46104  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  withdrawOnlyExcess           ·          -  ·          -  ·      61943  ·            1  ·          -  │
+|  VestingCvg           ·  withdrawOnlyExcess           ·          -  ·          -  ·      61919  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  WlPresaleCvg         ·  investMint                   ·     295021  ·     317287  ·     307430  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -91,11 +91,11 @@
 ························································|·············|·············|·············|···············|··············
 |  CvgControlTower                                      ·          -  ·          -  ·     859604  ·        2.9 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  CvgOracle                                            ·          -  ·          -  ·    1905520  ·        6.4 %  ·          -  │
+|  CvgOracle                                            ·          -  ·          -  ·    1947502  ·        6.5 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  CvgV3Aggregator                                      ·          -  ·          -  ·     620882  ·        2.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  Ibo                                                  ·    3041083  ·    3041095  ·    3041089  ·       10.1 %  ·          -  │
+|  Ibo                                                  ·    3133486  ·    3133498  ·    3133492  ·       10.4 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  mock_CloneFactoryV2                                  ·          -  ·          -  ·     535562  ·        1.8 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     665377  ·     720406  ·     692890  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1822493  ·        6.1 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1898072  ·        6.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2976839  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

For this reason, I would recommend implementing this change only if deployment costs are not a priority and you would rather have optimized runtime code.

Assuming that experienced users are calling the admin functions and that you want to make your admin functions payable regardless of the increased deployment costs, here is how this change would be implemented:

[CvgOracle.sol#L326](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L326)
```diff
- ) external onlyOwner {
+ ) external payable onlyOwner {
```

[CvgOracle.sol#L334](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L334)
```diff
- function setCvgToken(IERC20Metadata _cvg) external onlyOwner {
+ function setCvgToken(IERC20Metadata _cvg) external payable onlyOwner {
```

[Ibo.sol#L119](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L119)
```diff
- function createBond(BondParams calldata bondParams) external onlyOwner {
+ function createBond(BondParams calldata bondParams) external payable onlyOwner {
```

[Ibo.sol#L128](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L128)
```diff
- function setComposedFunction(uint256 bondId, uint8 newComposedFunction) external onlyOwner {
+ function setComposedFunction(uint256 bondId, uint8 newComposedFunction) external payable onlyOwner {
```

[VestingCvg.sol#L115](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L115)
```diff
- function setCvg(IERC20 _cvg) external onlyOwner {
+ function setCvg(IERC20 _cvg) external payable onlyOwner {
```

[VestingCvg.sol#L119](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L119)
```diff
- function setPresale(IPresaleCvgWl _newPresaleWl) external onlyOwner {
+ function setPresale(IPresaleCvgWl _newPresaleWl) external payable onlyOwner {
```

[VestingCvg.sol#L123](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L123)
```diff
- function setPresaleSeed(IPresaleCvgSeed _newPresaleSeed) external onlyOwner {
+ function setPresaleSeed(IPresaleCvgSeed _newPresaleSeed) external payable onlyOwner {
```

[VestingCvg.sol#L127](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L127)
```diff
- function setWhitelistTeam(address newWhitelistedTeam) external onlyOwner {
+ function setWhitelistTeam(address newWhitelistedTeam) external payable onlyOwner {
```

[VestingCvg.sol#L131](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L131)
```diff
- function setWhitelistDao(address newWhitelistedDao) external onlyOwner {
+ function setWhitelistDao(address newWhitelistedDao) external payable onlyOwner {
```

[VestingCvg.sol#L178](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L178)
```diff
- ) external onlyOwner {
+ ) external payable onlyOwner {
```

[VestingCvg.sol#L220](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L220)
```diff
- function revokeVestingSchedule(uint256 _vestingScheduleId) external onlyOwner {
+ function revokeVestingSchedule(uint256 _vestingScheduleId) external payable onlyOwner {
```

[VestingCvg.sol#L402](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L402)
```diff
- function withdrawOnlyExcess(uint256 _amount) external onlyOwner {
+ function withdrawOnlyExcess(uint256 _amount) external payable onlyOwner {
```

## [G-03] Write gas-optimal for-loops

### Instances

1. [Ibo.sol#L313-L315](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L313-L315)

### Issue

The for-loop mentioned above is not written in a gas-optimal way.

More specifically, it uses post-increment instead of pre-increment and implicitly checks the increment for a potential overflow although an overflow isn't possible.

### Recommendation

A gas-optimal for-loop looks like this:

```solidity
for (uint256 i; i < limit; ) {
    
    // inside the loop
    
    unchecked {
        ++i;
    }
}
```

The two differences here from a conventional for loop are that `i++` becomes `++i`, and that the loop variable's increment is unchecked because the `limit` variable ensures it won’t overflow. First, `++i` and `i++` are equivalent operations in the for-loop's logic, but `++i` is more gas-efficient than `i++`. Second, placing the increment inside `unchecked` will skip the superfluous check for an arithmetic overflow.

In conclusion, one can implement the following changes to save gas:

[Ibo.sol#L313-L315](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L313-L315)
```diff
- for (uint256 i = 0; i < range; i++) {
-     tokenIds[i] = tokenOfOwnerByIndex(_wallet, i);
- }
+ for (uint256 i; i < range; ) {
+     tokenIds[i] = tokenOfOwnerByIndex(_wallet, i);
+
+     unchecked {
+         ++i;
+     } 
+ } 
```

Implementing these changes and running the Hardhat gas reporter, one finds that the optimized loop saves 2160 gas.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index af0346b..42c8232 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -95,7 +95,7 @@
 ························································|·············|·············|·············|···············|··············
 |  CvgV3Aggregator                                      ·          -  ·          -  ·     622164  ·        2.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  Ibo                                                  ·    3101956  ·    3101968  ·    3101962  ·       10.3 %  ·          -  │
+|  Ibo                                                  ·    3099796  ·    3099808  ·    3099802  ·       10.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  mock_CloneFactoryV2                                  ·          -  ·          -  ·     535562  ·        1.8 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
```

## [G-04] Pre-increment cost less gas than post-increment

### Instances

1. [VestingCvg.sol#L213](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L213)

### Issue

A post-increment (`i++`) is more expensive in terms of gas than a pre-increment (`++i`). The reason is that `i++` increments the value of `i` but returns the original value. This means the EVM needs to temporarily store the original value before incrementing `i`. In the case of `++i`, however, the EVM can increment the value of `i` right away and simply return the incremented value; no need for temporary storage.

So, roughly speaking, the EVM will operate as follows:

`i++`:
1. Store the current value of `i` temporarily.
2. Increment `i`.
3. Return the stored (original) value.

`++i`:
1. Increment `i`.
2. Return the new value of `i`.

In particular, this means that pre- and post-increment are generally *not* equivalent operations. For example, doing

```solidity
i = 0;
array[i++];
```

will increment `i` by 1 but access the array at index 0. On the contrary, using `array[++i]` will increment `i` by 1, too, but access the array at index 1.

So, depending on the context, one *cannot* always replace `i++` with `++i` to save gas without changing the contract's logic.

Let me emphasize that the codebase provided for this audit contains two instances of post-increment, which should *not* be changed to pre-increment because this would change the logic due to the different return values. These instances are:

[Ibo.sol#L120](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L120)

[Ibo.sol#L169](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L169)

### Recommendation

There are two instances where the post-increment is used to increment a number but, at the same time, the post-increment's return value isn't used in the contract's logic. The first instance is the for-loop variable, which I already described in detail in another finding. The second instance can be found in `VestingCvg.sol`. Therefore, I recommend adjusting the code as follows:

[VestingCvg.sol#L213](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L213)
```diff
- nextVestingScheduleId++;
+ ++nextVestingScheduleId;
```

According to the Hardhat gas reporter, implementing this change will save 14 gas with each function call to `createVestingSchedule` and 1068 gas during deployment:

```diff
diff --git a/gasReporting.md b/gasReporting.md
index 42c8232..e0275b6 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -41,7 +41,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  SeedPresaleCvg       ·  setSaleState                 ·          -  ·          -  ·      28801  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  createVestingSchedule        ·     156058  ·     190318  ·     169769  ·           10  ·          -  │
+|  VestingCvg           ·  createVestingSchedule        ·     156044  ·     190304  ·     169755  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseIbo                   ·      71258  ·     123618  ·      84878  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     670502  ·     725531  ·     698015  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1852452  ·        6.2 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1851384  ·        6.2 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2977045  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

## [G-05] Cache storage variables

### Instances

1. [VestingCvg.sol#L171-L214](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L171-L214)

### Issue

Reading from a storage variable costs at least 100 gas as Solidity does not cache the storage read. Writes are considerably more expensive. Therefore, you should manually cache the variable and, ideally, only do exactly one storage read and exactly one storage write.

In the above-mentioned instance, a storage variable isn't cached but instead read from contract storage multiple times within the same function.

### Recommendation

Work with the cached value instead of reading from storage multiple times:

[VestingCvg.sol#L201-L213](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L201-L213)
```diff
- vestingSchedules[nextVestingScheduleId] = VestingSchedule({
-    revoked: false,
-    totalAmount: _totalAmount,
-    totalReleased: 0,
-    startTimestamp: _startTimestamp,
-    daysBeforeCliff: _daysBeforeCliff,
-    daysAfterCliff: _daysAfterCliff,
-    vestingType: _vestingType,
-    dropCliff: _dropCliff
- });
-  
- vestingIdForType[_vestingType] = nextVestingScheduleId;
- nextVestingScheduleId++;
+ vestingSchedules[vestingScheduleId] = VestingSchedule({
+    revoked: false,
+    totalAmount: _totalAmount,
+    totalReleased: 0,
+    startTimestamp: _startTimestamp,
+    daysBeforeCliff: _daysBeforeCliff,
+    daysAfterCliff: _daysAfterCliff,
+    vestingType: _vestingType,
+    dropCliff: _dropCliff
+ });
+  
+ vestingIdForType[_vestingType] = vestingScheduleId;
+ nextVestingScheduleId = vestingScheduleId + 1;
```
Notice that, should you decide to implement the above change, this will get rid of any pre-/post-increment operation that was either kept or implemented due to my recommendation regarding pre-/post-increments.

According to the Hardhat gas reporter, implementing these changes will save 297 gas per function call and a total of 6897 gas during deployment.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index e0275b6..5b82305 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -27,7 +27,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  CvgOracle            ·  setTokenOracleParams         ·      71411  ·      71675  ·      71540  ·           31  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59526  ·           22  ·          -  │
+|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59525  ·           22  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  ERC20                ·  approve                      ·      26558  ·      46700  ·      44526  ·          119  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -41,7 +41,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  SeedPresaleCvg       ·  setSaleState                 ·          -  ·          -  ·      28801  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  createVestingSchedule        ·     156044  ·     190304  ·     169755  ·           10  ·          -  │
+|  VestingCvg           ·  createVestingSchedule        ·     155747  ·     190007  ·     169458  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseIbo                   ·      71258  ·     123618  ·      84878  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     670502  ·     725531  ·     698015  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1851384  ·        6.2 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1844487  ·        6.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2977045  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

## [G-06] Constants can be private

### Instances

1. [BondCalculator.sol#L19](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Bond/BondCalculator.sol#L19)
2. [Ibo.sol#L65-L80](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L65-L80)
3. [VestingCvg.sol#L62-L72](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L62-L72)

### Issue

Some constants could be private but aren't.

Marking constants as private saves gas upon deployment, as the compiler does not have to create getter functions for these variables. It is worth noting that a private variable can still be read using either the verified contract source code or the bytecode. For immutable variables written via constructor parameters, you can also look the contract deployment transaction.

### Recommendation

Change the visibility modifier of the following constants to `private`:

[BondCalculator.sol#L19](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Bond/BondCalculator.sol#L19)
```diff
- uint256 public constant TEN_POWER_6 = 10 ** 6;
+ uint256 private constant TEN_POWER_6 = 10 ** 6;
```

[Ibo.sol#L65-L80](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L65-L80)
```diff
- address public immutable treasuryBonds;
- 
- /// @dev bond ROI calculator
- IBondCalculator public immutable bondCalculator;
-
- /// @dev oracle fetching prices in LP
- ICvgOracle public immutable cvgOracle;
-
- uint256 public immutable iboStartTimestamp;
-
- /// @dev 0.33$
- uint256 public constant CVG_PRICE_NO_ROI = 33 * 10 ** 16;
- uint256 public constant IBO_DURATION = 6 hours;
- 
- uint256 public constant MAX_CVG_PEPE_PRIVILEGE = 15_000 * 10 ** 18;
- uint256 public constant MAX_CVG_WL_PRIVILEGE = 7_500 * 10 ** 18;
+ address private immutable treasuryBonds;
+ 
+ /// @dev bond ROI calculator
+ IBondCalculator private immutable bondCalculator;
+
+ /// @dev oracle fetching prices in LP
+ ICvgOracle private immutable cvgOracle;
+
+ uint256 private immutable iboStartTimestamp;
+
+ /// @dev 0.33$
+ uint256 public constant CVG_PRICE_NO_ROI = 33 * 10 ** 16;
+ uint256 private constant IBO_DURATION = 6 hours;
+ 
+ uint256 private constant MAX_CVG_PEPE_PRIVILEGE = 15_000 * 10 ** 18;
+ uint256 private constant MAX_CVG_WL_PRIVILEGE = 7_500 * 10 ** 18;
```

Notice that the above `CVG_PRICE_NO_ROI` has been left public because its implicit getter function is directly accessed in one of the tests.

[VestingCvg.sol#L62-L72](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L62-L72)
```diff
- uint256 public constant MAX_SUPPLY_TEAM = 12_750_000 * 10 ** 18;
- uint256 public constant MAX_SUPPLY_DAO = 15_000_000 * 10 ** 18;
-
- uint256 public constant ONE_DAY = 1 days;
-
- uint256 public constant ONE_GWEI = 10 ** 9;
+ uint256 private constant MAX_SUPPLY_TEAM = 12_750_000 * 10 ** 18;
+ uint256 private constant MAX_SUPPLY_DAO = 15_000_000 * 10 ** 18;
+
+ uint256 private constant ONE_DAY = 1 days;
+
+ uint256 private constant ONE_GWEI = 10 ** 9;
```

From the output of the Hardhat gas reporter, we can see that the function calls are 88 gas more expensive (on average, and assuming all functions are called equally often). However, one saves a total of 84869 gas during deployment.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index 46c6f97..ca87ff6 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -31,7 +31,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  ERC20                ·  approve                      ·      26558  ·      46700  ·      44526  ·          119  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  Ibo                  ·  createBond                   ·          -  ·          -  ·      98432  ·            7  ·          -  │
+|  Ibo                  ·  createBond                   ·          -  ·          -  ·      98410  ·            7  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  Ibo                  ·  deposit                      ·     112237  ·     320062  ·     240553  ·           23  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -43,7 +43,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  createVestingSchedule        ·     155747  ·     190007  ·     169458  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  releaseIbo                   ·      71258  ·     123618  ·      84878  ·            4  ·          -  │
+|  VestingCvg           ·  releaseIbo                   ·      71280  ·     123640  ·      84900  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseSeed                  ·      73648  ·     126008  ·      91384  ·            5  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -53,17 +53,17 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  revokeVestingSchedule        ·          -  ·          -  ·      38463  ·            2  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setCvg                       ·          -  ·          -  ·      46194  ·            1  ·          -  │
+|  VestingCvg           ·  setCvg                       ·          -  ·          -  ·      46172  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setPresale                   ·          -  ·          -  ·      29006  ·            1  ·          -  │
+|  VestingCvg           ·  setPresale                   ·          -  ·          -  ·      29094  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setPresaleSeed               ·          -  ·          -  ·      28994  ·            1  ·          -  │
+|  VestingCvg           ·  setPresaleSeed               ·          -  ·          -  ·      29016  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  setWhitelistDao              ·          -  ·          -  ·      46149  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  setWhitelistTeam             ·          -  ·          -  ·      46150  ·            1  ·          -  │
+|  VestingCvg           ·  setWhitelistTeam             ·          -  ·          -  ·      46128  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  withdrawOnlyExcess           ·          -  ·          -  ·      61921  ·            1  ·          -  │
+|  VestingCvg           ·  withdrawOnlyExcess           ·          -  ·          -  ·      61943  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  WlPresaleCvg         ·  investMint                   ·     295021  ·     317287  ·     307430  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -83,7 +83,7 @@
 ························································|·············|·············|·············|···············|··············
 |  BaseTest                                             ·          -  ·          -  ·     388889  ·        1.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  BondCalculator                                       ·          -  ·          -  ·    1081664  ·        3.6 %  ·          -  │
+|  BondCalculator                                       ·          -  ·          -  ·    1077092  ·        3.6 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  CloneFactory                                         ·          -  ·          -  ·     582766  ·        1.9 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
@@ -95,7 +95,7 @@
 ························································|·············|·············|·············|···············|··············
 |  CvgV3Aggregator                                      ·          -  ·          -  ·     622164  ·        2.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  Ibo                                                  ·    3099796  ·    3099808  ·    3099802  ·       10.3 %  ·          -  │
+|  Ibo                                                  ·    3041289  ·    3041301  ·    3041295  ·       10.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  mock_CloneFactoryV2                                  ·          -  ·          -  ·     535562  ·        1.8 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     670502  ·     725531  ·     698015  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1844487  ·        6.1 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1822697  ·        6.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2977045  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

## [G-07] Use Solidity 0.8.19 or higher to save gas

### Instances

1. [BondCalculator.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Bond/BondCalculator.sol#L12)
2. [CvgOracle.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L12)
3. [Ibo.sol#L24](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L24)
4. [VestingCvg.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L12)

### Issue

The Solidity release 0.8.19 introduced some changes that will make the same code run more gas-efficiently compared to versions below 0.8.19.

### Recommendation

Replace the current version pragmas with 0.8.19 or higher and compile the contracts with the new version. In all four instances

[BondCalculator.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Bond/BondCalculator.sol#L12)

[CvgOracle.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/Oracles/CvgOracle.sol#L12)

[Ibo.sol#L24](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/Ibo.sol#L24)

[VestingCvg.sol#L12](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L12)

the change to the version pragma 0.8.19 would look like this:

```diff
- pragma solidity ^0.8.0;
+ pragma solidity 0.8.19;
```

Notice that the new version pragma is no longer "floating", i.e., it only allows for compilation with only one specific compiler version. 

Assuming you use Solidity 0.8.19, the output of the Hardhat gas reporter tells us that these changes will save 23 gas on the function calls and a total of 5535 gas during deployment.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index a71dda1..f1a226e 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -1,5 +1,5 @@
 ·-------------------------------------------------------|---------------------------|-------------|-----------------------------·
-|                 Solc version: 0.8.17                  ·  Optimizer enabled: true  ·  Runs: 250  ·  Block limit: 30000000 gas  │
+|                 Solc version: 0.8.19                  ·  Optimizer enabled: true  ·  Runs: 250  ·  Block limit: 30000000 gas  │
 ························································|···························|·············|······························
 |  Methods                                                                                                                      │
 ························|·······························|·············|·············|·············|···············|··············
@@ -7,7 +7,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  BaseTest             ·  incrementCounter             ·          -  ·          -  ·      48258  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CloneFactory         ·  createCvgAggregator          ·     201615  ·     218835  ·     204161  ·           14  ·          -  │
+|  CloneFactory         ·  createCvgAggregator          ·     201603  ·     218823  ·     204149  ·           14  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  Cvg                  ·  approve                      ·          -  ·          -  ·      46596  ·            2  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -27,7 +27,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  CvgOracle            ·  setTokenOracleParams         ·      71411  ·      71675  ·      71540  ·           31  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59525  ·           22  ·          -  │
+|  CvgV3Aggregator      ·  setLatestPrice               ·      37766  ·      71978  ·      59526  ·           22  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  ERC20                ·  approve                      ·      26558  ·      46700  ·      44526  ·          119  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -35,7 +35,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  Ibo                  ·  deposit                      ·     112237  ·     320063  ·     240553  ·           23  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  mock_CloneFactoryV2  ·  createBaseTest               ·          -  ·          -  ·     119967  ·            3  ·          -  │
+|  mock_CloneFactoryV2  ·  createBaseTest               ·          -  ·          -  ·     119955  ·            3  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  ProxyAdmin           ·  upgrade                      ·          -  ·          -  ·      38853  ·            1  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -81,29 +81,29 @@
 ························································|·············|·············|·············|···············|··············
-|  TransparentUpgradeableProxy                          ·     670502  ·     725531  ·     698015  ·        2.3 %  ·          -  │
+|  TransparentUpgradeableProxy                          ·     665377  ·     720406  ·     692890  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1822697  ·        6.1 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1822493  ·        6.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  WlPresaleCvg                                         ·          -  ·          -  ·    2977045  ·        9.9 %  ·          -  │
+|  WlPresaleCvg                                         ·          -  ·          -  ·    2976839  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

## [G-08] Consider choosing a very large value for the optimizer

The Solidity optimizer focuses on optimizing two primary aspects:

1. The deployment cost of a smart contract.
2. The execution cost of functions within the smart contract.

There’s a trade-off involved in selecting the runs parameter for the optimizer.

Smaller run values prioritize minimizing the deployment cost, resulting in smaller creation code but potentially unoptimized runtime code. While this reduces gas costs during deployment, it may not be as efficient during execution.

Conversely, larger values of the runs parameter prioritize the execution cost. This leads to larger creation code but an optimized runtime code that is cheaper to execute. While this may not significantly affect deployment gas costs, it can significantly reduce gas costs during execution.

Given this trade-off, if your contract will be used frequently, you should consider using a larger value for the optimizer, as this will save gas costs in the long term.

## [G-09] Split require statements

### Instances

1. [VestingCvg.sol#L179-L183](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L179-L183)

### Issue

There is a require statement whose first argument is a boolean `&&` expression and can, therefore, be split into two require statements.

When we split require statements, we are essentially saying that each statement must be true for the function to continue executing.

If the first statement evaluates to false, the function will revert immediately, and the following require statements will not be examined. This will save the gas cost rather than evaluating the next require statement.

### Recommendation

As we can see from the Hardhat gas reporter, making this change will optimize the cost of runtime code but increase the deployment costs significantly.

```diff
diff --git a/gasReporting.md b/gasReporting.md
index 40846af..3a5359d 100644
--- a/gasReporting.md
+++ b/gasReporting.md
@@ -41,7 +41,7 @@
 ························|·······························|·············|·············|·············|···············|··············
 |  SeedPresaleCvg       ·  setSaleState                 ·          -  ·          -  ·      28801  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
-|  VestingCvg           ·  createVestingSchedule        ·     155747  ·     190007  ·     169458  ·           10  ·          -  │
+|  VestingCvg           ·  createVestingSchedule        ·     155739  ·     189999  ·     169450  ·           10  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
 |  VestingCvg           ·  releaseIbo                   ·      71280  ·     123640  ·      84900  ·            4  ·          -  │
 ························|·······························|·············|·············|·············|···············|··············
@@ -103,7 +103,7 @@
 ························································|·············|·············|·············|···············|··············
 |  TransparentUpgradeableProxy                          ·     665377  ·     720406  ·     692890  ·        2.3 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
-|  VestingCvg                                           ·          -  ·          -  ·    1822493  ·        6.1 %  ·          -  │
+|  VestingCvg                                           ·          -  ·          -  ·    1837122  ·        6.1 %  ·          -  │
 ························································|·············|·············|·············|···············|··············
 |  WlPresaleCvg                                         ·          -  ·          -  ·    2976839  ·        9.9 %  ·          -  │
 ·-------------------------------------------------------|-------------|-------------|-------------|---------------|-------------·
```

For this reason, I would recommend implementing this change only if deployment costs are not a priority and you would rather have optimized runtime code.

If you do decide to split the require statements, here is how this change would be implemented:

[VestingCvg.sol#L179-L183](https://github.com/Cvg-Finance/hats-audit/blob/da48577d2f42fa8c2e35bb7223208ea6ba88012e/contracts/PresaleVesting/VestingCvg.sol#L179-L183)
```diff
- require(
-     presaleSeed.saleState() == IPresaleCvgSeed.SaleState.OVER &&
-         presaleWl.saleState() == IPresaleCvgWl.SaleState.OVER,
-     "PRESALE_ROUND_NOT_FINISHED"
- );
+ require(
+     presaleSeed.saleState() == IPresaleCvgSeed.SaleState.OVER,
+     "PRESALE_ROUND_NOT_FINISHED"
+ );
+ require(
+     presaleWl.saleState() == IPresaleCvgWl.SaleState.OVER,
+     "PRESALE_ROUND_NOT_FINISHED"
+ );
```
