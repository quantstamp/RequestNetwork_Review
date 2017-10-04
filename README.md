# Overview

This smart contract audit was prepared by [Quantstamp](https://www.quantstamp.com/), the protocol for securing smart contracts.

## Specification

Our understanding of the specification was based on the description of the token
sale provided in the [README.md](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/README.md) file in the github repository at the time of the
audit. We also reviewed the white-paper dated October 1, 2017.

## Methodology

The review was conducted during 2017-Oct-01 thru 2017-Oct-05 by the Quantstamp
team, which included senior engineers Vajih Montaghami, Ed Zulkoski and Steven
Stewart.

Their procedure can be summarized as follows:

1. Code review
    * Manual review of code
    * Comparison to specification
2. Testing and automated analysis
    * Test coverage analysis
    * Symbolic execution (automated code path evaluation)
    * Fuzzer for test-case generation
3. Best-practices review
4. Itemize recommendations

## Source Code

The following source code was reviewed during the audit.

| Repository                                                             | Commit                                                                                                    |
|------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| [RequestTokenSale](https://github.com/RequestNetwork/RequestTokenSale) | [61e5b](https://github.com/RequestNetwork/RequestTokenSale/tree/61e5b3366e12a9098eb9305887bc45e67a1e6150) |

# Security Audit

Quantstamp's objective was to evaluate the RequestNetwork code for security-related issues, code quality and adherence to best-practices.

Possible issues include (but are not limited to):

* Transaction-ordering dependence
* Timestamp dependence
* Mishandled exceptions and call stack limits
* Unsafe external calls
* Integer overflow / underflow
* Reentrancy and cross-function vulnerabilities
* Denial of service / logical oversights

We did not find any examples of the above issues, apart from one example of an addition involving timestamps that did not use the ```SafeMath``` library (line 67, [RequestTokenSale.sol](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/contracts/ProgressiveIndividualCappedCrowdsale.sol)). Although we are not concerned about timestamps approaching the max value of a 256-bit integer, we do recommend using the ```SafeMath``` library functions as a best-practice.

# Test coverage

We evaluated the test coverage using truffle and solidity-coverage. The below notes outline the setup and steps that were performed.

## Setup

Testing setup:
* Truffle v3.4.11
* TestRPC v4.1.3
* solidity-coverage v0.2.5

## Steps

Steps taken to run the full test suite:

* Removed the ```return;``` statements at the top of test/*.js. These were presumably added due to unwanted interactions between tests when running the full suite all at once.

* In order to ensure that the test files were executed in a deterministic manner, a chain of require statements among the tests were added. For example, we added the line ```require("./TransferToken.js");``` to creationCrowdsale.js, which ensures that TransferToken is run after creationCrowdsale.

* The state of the blockchain time data did not seem to update properly in between test files. This would cause the creation of RequestTokenSale to fail due to the startTime parameter not being greater than 'now'. In order for all tests to function properly, the ```addsDayOnEVM()``` function was called appropriately before each creation of a RequestTokenSale. This behavior may not exist with different versions of the aforementioned tools.

* There was one faulty test case that was due to a hard-coded date value. This test ensures that RequestTokenSale cannot be instantiated with an end-date before the start-date. The hard-coded date caused unforseen interaction with the actual current date, which presumably did not occur when the test was originally written. Upon updating the constant to an appropriate value the test passed as expected.

Additional steps taken to run solidity-coverage on the test suite:

* Include the testrpc launch parameters from launchTestrpc.sh in the .solcover configuration file.

## Evaluation

Based on both automated and manual analysis, we noted that test coverage was comprehensive. There were notable absences of coverage for contracts borrowed from Zeppelin-Solidity, but these are widely used and known to be reliable, and so we consider this gap in coverage acceptable.

Although we do not anticipate any issues, we did observe that the else path for numerous require statements were not covered by the existing tests. These include:

* Line 29, CappedCrowdsale.sol
* Line 29, 30, RequestToken.sol
* Line 49, 50, StandardCrowdsale.sol

For more complete coverage, it is desirable to include tests that exercise all code paths, including those for which reversions of state will be triggered when a call to ```require()``` is encountered.


# Recommendations

## Code documentation

We noted that some functions were well-documented, while others were not. We recommend the usage of documentation tags such as ```@dev```, ```@param``` and ```@returns``` for all functions.

## Visibility modifiers

We recommend always including visibility modifiers for functions and variables, even when they are public. On line 85 of [RequestToken.sol](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/contracts/RequestToken.sol), the visibility modifier for ```burnFrom()``` is missing, which should be ```public```.

## Valid purchase

The ```validPurchase()``` function is used to build a Boolean expression of contraints that must be satisified in order for a purchase to proceed. It is reasonable to expect such a function to be constant (i.e., unable to modify state). Unfortunately, the solidity compiler does not presently enforce the constant modifier.

In [ProgressiveIndividualCappedCrowdsale](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/contracts/ProgressiveIndividualCappedCrowdsale.sol), we observed that the ```participated``` mapping is modified within ```validPurchase()```.

        participated[msg.sender] = participated[msg.sender].add(msg.value);

From the strictest point of view, this seemingly breaks the expected behaviour of the constant function.

We recommend separating the check, which does not modify internal state, from the pre-purchase actions that do modify internal state.

## Pausable

Even when the probability of a bug is relatively low, the ability to trigger an emergency pause is a recommended precaution. An example contract and modifier can be found [here](https://github.com/OpenZeppelin/zeppelin-solidity/blob/master/contracts/lifecycle/Pausable.sol) at Zeppelin-Solidity.

## Check return values

We recommend always checking the return value of a function call. On line 92 of [StandardCrowdsale.sol](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/contracts/RequestTokenSale.sol), an unchecked call to ```token.transfer()``` is made.

We acknowledge that, presently, the transfer function will either return ```true``` or trigger a reversion of state; however, it is possible that unanticipated future changes to the code may result in a scenario when ```false``` is returned. By checking the return value now, a possible future bug may be avoided.

## Hard-coded decimal value

In general, it is desirable to avoid hard-coded values. The number of decimals specified for the token is 18, and this value is hard-coded when computing the values of constants in the crowdsale contract. If there were a future decision to change the value, it would need to be modified in multiple places, increasing the probability of a bug.

We note that the interaction between crowdsale and token makes it so that it is not possible to access the storage of the token when declaring the constants of the crowdsale. In this design, it is necessary to hard code the number of decimals when computing the token allocations. An alternative design in which the token contract computes the token allocations and passes them to the constructor of the crowdsale could rectify this seemingly benign issue.

We also acknowledge that it is unlikely that Request.Network will change this value, and that the correct value of 18 was used throughout. We think it is acceptable to use the current approach in spite of highlighting this commonly held best practice.

## Dead code

In [StandardCrowdsale.sol](), the modifier ```onlyBeforeSale``` does not appear to be used and could (optionally) be removed.

## Compiler warnings

The solidity compiler reported three warnings about hard-coded addresses in [RequestTokenSale.sol](https://github.com/RequestNetwork/RequestTokenSale/blob/61e5b3366e12a9098eb9305887bc45e67a1e6150/contracts/ProgressiveIndividualCappedCrowdsale.sol). To clear these warnings, use checksummed addresses.

Reference: [how to specify a hard-coded address as a literal](https://ethereum.stackexchange.com/questions/17060/solidity-how-to-specify-a-hard-coded-address-as-a-literal/17065)


# Appendix

## File Signatures

Below are SHA256 file signatures of the relevant files reviewed in the audit.

```
$ shasum -a 256 ./contracts/*
8b09b9c8e58b1d3f3e4b7f073620a0290602aa1628d27045d55c6bbdb497b8dc  ./contracts/Migrations.sol
68950e2528b86adc253f1ec414a5b7b6b9ea1c07ed1d17ef942280dbf2dbace1  ./contracts/ProgressiveIndividualCappedCrowdsale.sol
fd2f185f39a47b361186ae7c521c6ddfb3e2aaa909fc9af47fc54acc30e1f873  ./contracts/RequestToken.sol
71cd7bb7ef31bb66ccf37b42121081fc4cf8b1aff13a0b2fbe39b25e822d342a  ./contracts/RequestTokenSale.sol
a8f0a0a48d1bf02de3a9d25dc8d2501209865bf229e196ddadd26f8f32613238  ./contracts/StandardCrowdsale.sol
409ffaa24d8c18774b236edeeda7e976193002520b343e41d998b7d189cfe177  ./contracts/WhitelistedCrowdsale.sol

$ shasum -a 256 ./test/*
c89a05f0d5f6215c32958f9c01f37d527ee63b80f3cf673a08db0012b40915e2  ./test/BuyCrowdsale.js
ec65b354442335d623248831d02026fdbcc9281bf933550252c3e4be952d8044  ./test/DrainTokenEndCrowdsale.js
c8e251e22489c83bd1330fdb9572f541f0e651e10e567d20cef61ba0a89db96a  ./test/TimeTestTokenSale.js
360978bb6ec1a120751da4bbadc847eff1d02871659fe84fd5c0f99aa69d4346  ./test/TransferToken.js
08deb44ca126e9ff3b927ea6efc33fbbe156427a31823489098a81c0a9e3c349  ./test/WhiteListCrowdsale.js
373da1e4a8272ec36009396dd84a1fa6e8c8a7a2488bec4033460295c0219785  ./test/creationCrowdsale.js

$ shasum -a 256 ./migrations/*
42c21b4229b39fd1cad164ed6d4c24168620e2f04a66521b2b2f2945e23b867d  ./migrations/1_initial_migration.js
5809b0ebd7c0929d665dddd6ef5532282b9925d7ae1ccaf7173244a5f84358ce  ./migrations/2_deploy_contracts.js
```

# Disclosure

## Purpose of report

The scope of our review is limited to a review of Solidity code and only the source code we note as being within the scope of our review within this report. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks.

The report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.

No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project.

## Links to other websites
You may, through hypertext or other computer links, gain access to web sites operated by persons other than Quantstamp Technologies Inc. (QTI). Such hyperlinks are provided for your reference and convenience only, and are the exclusive responsibility of such web sites' owners. You agree that QTI are not responsible for the content or operation of such web sites, and that QTI shall have no liability to you or any other person or entity for the use of third-party web sites. Except as described below, a hyperlink from this web site to another web site does not imply or mean that QTI endorses the content on that web site or the operator or operations of that site. You are solely responsible for determining the extent to which you may use any content at any other web sites to which you link from the report. QTI assumes no responsibility for the use of third-party software on the website and shall have no liability whatsoever to any person or entity for the accuracy or completeness of any outcome generated by such software.

## Timeliness of content
The content contained in the report is current as of the date appearing on the report and is subject to change without notice, unless indicated otherwise by QTI; however, QTI does not guarantee or warrant the accuracy, timeliness, or completeness of any report you access using the internet or other means, and assumes no obligation to update any information following publication.