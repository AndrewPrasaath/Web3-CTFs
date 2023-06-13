# List of Challenges
1. [Fallback challenge](#1-fallback)
# 1. Fallback
### Challenge
- Claim ownership of the contract
- Reduce its balance to 0
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
2. Send `1 wei` to the contract address. Ownership is transferred.\
\
Congratulations! We successfully became the owner of the `Fallback contract`.
