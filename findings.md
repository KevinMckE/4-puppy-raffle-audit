### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incrementing gas costs for future entrants

Impact: MEDIUM
LIKELIHOOD: MEDIUM

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::players` array is, the more checks a new player will have to make. This means the gas costs for players who enter right when the raffle starts will be dramatically lower than those who enter later. Every additional address in the `players` array, is an additional check the loop will have to make, and eventually this will increase the gas costs so much that there will be an EVM reversion.

**Impact:** The gas costs for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from enteringm, and causing a rush at the start of a raffle to be one of the first entrants in the queue. 

An attacker mighe make the `PuppyRaffle::entrants` array so big, that no one else enters, guaranteeing themselves the win.

**Proof of Concept:**

If 1,000 players enter, the gas cost eventually balloons up to `1056944034`, causing an `EvmError: Revert`.

<details>
<summary> PoC </summary>

Place the following test into `PuppyRaffleTest.t.sol`.

``` javascript
function testDenialOfService() public {
        for (uint256 i = 0; i < 1000; i++) {
            address[] memory players = new address[](i);
            puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        }
    }
```

Output: 

```
Failing tests:
Encountered 1 failing test in test/PuppyRaffleTest.t.sol:PuppyRaffleTest
[FAIL: EvmError: Revert] testDenialOfService() (gas: 1056944034)
```

</details>

**Recommended Mitigation:** There are a few options for recommended mitigations

1. Consider allowing duplicates. Users can make new wallet adresses anyways, so a duplicate ch3eck doesn't prevent the same person from entering from multple wallet addresses.
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of wheter a user has already entered.

```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;
        }
​
-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

3. Alternatively, you could use OpenZeppelin's EnumerableSet library.
(https://docs.openzeppelin.com/contracts/5.x/api/utils#EnumerableSet)