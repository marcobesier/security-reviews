# Beanstalk Security Review

This security review shows a finding I submitted as part of the Beanstalk contest on CodeHawks in April 2024. I tried to submit this one as a Low, but it was judged as informational. All of my other findings were already listed as known issues and, therefore, not submitted.

|   Severity   |   Issue   | 
|--------------|-----------|
|[I-01]| Vulnerable Solidity version in use | 

## [I-01] Vulnerable Solidity version in use

### Description

For all contracts in scope, an outdated Solidity version, 0.7.6, is currently in use. However, at the time of writing, there are [9 known bugs](https://docs.soliditylang.org/en/latest/bugs.html) in version 0.7.6.

### Impact

This can introduce vulnerabilities to the project that were fixed in newer solidity versions, such as the [Head Overflow Bug in Calldata Tuple ABI-Reencoding](https://blog.soliditylang.org/2022/08/08/calldata-tuple-reencoding-head-overflow-bug/).

### Recommended Mitigation

Consider upgrading the codebase to use a version of Solidity that does not contain known bugs, e.g., 0.8.23.
