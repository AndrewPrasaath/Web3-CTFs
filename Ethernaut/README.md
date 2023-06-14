# Challenges
1. [Fallback](#1-fallback)
2. [Fallout](#2-fallout)
3. [Coin Flip](#3-coin-flip)
# 1. Fallback
### Challenge
- Claim ownership of the contract.
- Reduce its balance to 0.
### Purpose
To know the basics of how ether goes in and out of contracts, including the usage of the fallback method.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```
### Solution
##### Explanation
**Receive function:** It executes on calls to the contract with no calldata, e.g. calls made via send() or transfer(). This is a function triggered when a contract receives ether without any calldata.\
\
How can we exploit this? There are two ways to claim ownership here: contribute to the contract more than the current owner (or) exploit the faulty check-in receive function. If we have any value of contribution and send any amount of ether to the contract, we will claim the ownership.
##### Exploit
All we need is a test ETH of 2 wei for the contract and the required test ETH for gas. I used `Remix IDE` (other options Etherscan, Hardhat, etc).
1. Invoke `contribute()` function with 1 wei. This will satisfy `contributions[msg.sender] > 0` condition
2. Send `1 wei` to the contract address. Ownership is transferred.
3. Since ownership is changed we can access `withdraw()` with `onlyOwner` modifier. Call `withdraw()` to drain the contract balance.\
\
Congratulations! We successfully became the owner of the `Fallback contract` and stole the balance in it.

# 2. Fallout
### Challenge
- Claim ownership of the contract
### Purpose
Keep upto date with solidity updates.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```
### Solution
##### Explanation
**Solidity v0.5.0 Breaking Changes:** Constructor needs to be defined using the `constructor` keyword instead defining a function with the same name as the contract. [solidity doc link](https://docs.soliditylang.org/en/v0.8.20/050-breaking-changes.html#constructors)\
Solidity compiler throws an error if there is a function with the name of the contract. That's the reason here they misspelled the `Fal1out` for `Fallout`. Since this uses a version above 0.5.0, the `constructor` keyword must be used for the constructor. This makes the function `Fal1out` just a normal function that can be called by anyone.
##### Exploit
1. Call `Fal1out()` and the ownership is changed.\
\
Hurray! We got another ownership.

# 3. Coin Flip
### Challenge
- Predict the outcome coin flip 10 times in a row.
### Purpose
Understanding generating random numbers in solidity and their potential threat. 
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```
### Solution
##### Explanation
There currently isn't a native way to generate them, and everything we use in smart contracts is publicly visible, including the local variables and state variables marked as private. Miners also have control over things like block hashes, timestamps, and whether to include certain transactions - which allows them to bias these values in their favor.\
I create another contract to attack `CoinFlip` by exploiting the same logic and factor used in it. Making an external call from an attack contract sends one transaction so that both will have the same block information.
##### Exploit
```
import "./CoinFlip.sol"

contract AttackCoinFlip {
    CoinFlip public target;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(CoinFlip _target) {
        target = _target; // instance of the CoinFlip contract to be attacked
    }

    // Attack function
    function attack() public {
        // Logic implementation used in flip() of CoinFlip
        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool guess = coinFlip == 1 ? true : false;
        // Attacking contract
        bool attackSuccess = target.flip(guess);
        require(attackSuccess, "Attack failed");
    }
}
```
1. Deploy the `AttackCoinFlip` contract with `CoinFlip` address as an argument.
2. Invoke `attack` 10 times to increase `consecutiveWins` in CoinFlip (check whether `consecutiveWins` has a value minimum of 10).\
\
Submit the contract instance on Ethernaut to claim the win.\
To get cryptographically proven random numbers, we can use Chainlink VRF, which uses an oracle, the LINK token, and an on-chain contract to verify that the number is truly random.\
Some other options include using Bitcoin block headers (verified through BTC Relay, RANDAO, or Oraclize).
