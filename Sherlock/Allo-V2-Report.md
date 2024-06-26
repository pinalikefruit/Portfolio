# [Allo V2](https://audits.sherlock.xyz/contests/109)

| Severity | Title |  Report Link |
|:--:|:---|:---|
| H-01 | The `Anchor.sol` sets up the registry with an incorrect address| _[Report-H01](https://github.com/sherlock-audit/2023-09-Gitcoin-judging/issues/335)_ |


## [H-01] The `Anchor.sol` sets up the registry with an incorrect address

## Summary
When `Anchor.sol` was deployed and the registry is configured with a wrong address. Therefore, the owner will never be able to call `Anchor.execute()` correctly, because it will always fail, since the function does not exist for the implemented address and if someone sends funds they will be lost forever.

## Vulnerability Detail
When the owner tries to create a profile using `Registry.createProfile()` in the `Registry.sol`, the `_generateAnchor()` function is called internally and `CREATE3` is implemented using the solady library to deploy the Anchor associated with the owner's profileId.

This implementation works fine so far. Then, when the Anchor.sol contract is deployed, the constructor is called. Next, this statement is executed within the constructor:

``` registry = Registry(msg.sender);```

The protocol expected `msg.sender` to be `Registry.sol`, since the deploy call is made by `Registry.sol`. But this address is not. But rather the temporary creation context created by the `CREATE3.deploy` function.

Therefore, the address of Registry in `Anchor.sol` is neither the Implementer nor the EOA of the owner caller.

You can prove this is correct using this test:

```solidity
pragma solidity 0.8.19;

import "forge-std/Test.sol";
import {Anchor} from "../../../contracts/core/Anchor.sol";
import {Registry} from "../../../contracts/core/Registry.sol";
import {Metadata} from "../../../contracts/core/libraries/Metadata.sol";

contract RegistryTest is Test {

    function setUp() public {
        registry = new Registry();
        registry.initialize(msg.sender);
    }

    function test_isNotSetRegistry() public {
        vm.deal(bob,1000 ether);
        vm.startPrank(bob);
        Metadata memory metadata = Metadata({protocol: 1, pointer: "test metadata"});
        address[] memory emptyArray = new address[](0);
        bytes32 profileId = registry.createProfile(0,"test", metadata,address(bob),emptyArray);
        Registry.Profile memory profile = registry.getProfileById(profileId);
        address  payable anchorAddress = payable(profile.anchor);
        anchor = Anchor(anchorAddress);
        
        assertTrue(address(registry) != address((anchor.registry())));
    }
}

```
Giving confirmed proof that the addresses are not the same.

## Impact
`Anchor.sol` can never be used, funds can be received and how do they have the restriction of verifying if it is the owner and that person calls `Registry.sol`. Since it is not configured correctly it is never successful for the `Anchor.execute()` call. Thus, the funds will always remain blocked and will be lost.

The protocol logic for this implementation will not work.

The team called this contract a crucial utility within the Allo ecosystem, making it easier to execute calls to destination addresses. Noting how serious this issue is.

## Code Snippet
https://github.com/sherlock-audit/2023-09-Gitcoin/blob/main/allo-v2/contracts/core/Anchor.sol#L56

## Tool used
* Manual Review
* Foundry


## Recommendation
Consider explicitly passing them the registry address as arguments.

This way you can ensure that the registry has the absolutely correct address.
```diff 
-   constructor(bytes32 _profileId) {
+   constructor(bytes32 _profileId, address registryAddress)
-     registry = Registry(msg.sender);
+     registry = Registry(registryAddress);
        profileId = _profileId;
    }
```