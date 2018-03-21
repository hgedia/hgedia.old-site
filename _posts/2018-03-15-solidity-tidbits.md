---
title: "Solidity tidbits"
categories:
  - Ethereum
tags:
  - opcodes
  - contract
layout: post_custom  
---
The following post contains tidbits I found interesting and good to be aware of while working with solidity.


{% include toc %}
{:toc}

# Callcode vs Delegate call

Call , Callcode and Delegate call differ in terms of what storage and msg.sender is used in the contract.

Ex: 
Contract : 

```
contract D {
  uint public n;
  address public sender;

  function callSetN(address _e, uint _n) {
    _e.call(bytes4(sha3("setN(uint256)")), _n); // E's storage is set, D is not modified 
  }

  function callcodeSetN(address _e, uint _n) {
    _e.callcode(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified 
  }

  function delegatecallSetN(address _e, uint _n) {
    _e.delegatecall(bytes4(sha3("setN(uint256)")), _n); // D's storage is set, E is not modified 
  }
}

contract E {
  uint public n;
  address public sender;

  function setN(uint _n) {
    n = _n;
    sender = msg.sender;
    // msg.sender is D if invoked by D's callcodeSetN. None of E's storage is updated
    // msg.sender is C if invoked by C.foo(). None of E's storage is updated

    // the value of "this" is D, when invoked by either D's callcodeSetN or C.foo()
  }
}

contract C {
    uint public n;
    function foo(D _d, E _e, uint _n) {
        _d.delegatecallSetN(_e, _n);
    }
    
    function fooDelegateCall(D _d, E _e, uint _n) {
        _d.delegatecall(bytes4(sha3("delegatecallSetN(address,uint256)")),_e,_n);
    }    
}
```

Assume deployed addresses : 

```
C : 0x111111111111
D : 0x222222222222
E : 0x333333333333
Account sending tx : 0x444444444444
```

<h3> Call</h3>
```
d.callSetN('0x333333333333',20);
```

Storage   : Value is set in the storage at location 0x333333333333 <i>[Contract E]</i> to 20  <br>
msg.sender: Value is 0x222222222222  <i>[Contract D]</i>

<h3> Callcode</h3>
```
d.callcodeSetN('0x333333333333',50);
```

Storage   : Value is set in the storage at location 0x222222222222 <i>[Contract D]</i> to 50  <br>
msg.sender: Value is 0x222222222222 <i>[Contract D]</i>

<h3> Delegatecall</h3>
```
d.delegatecallSetN('0x333333333333',70);
```
Storage   : Value is set in the storage at location 0x222222222222 <i>[Contract D]</i> to 70  <br>
msg.sender: Value is 0x444444444444 <i>[Sending Account]</i>


<h3> Delegatecall pt.2</h3>
```
c.foo('0x222222222222','0x333333333333',120);
```
Storage   : Value is set in the storage at location 0x222222222222 <i>[Contract D]</i> to 120 <br>
msg.sender: Value is 0x222222222222 <i>[Contract D]</i>


<h3> Delegatecall pt.3</h3>
```
c.fooDelegatecall('0x222222222222','0x333333333333',140);
```
Storage   : Value is set in the storage at location 0x111111111111 <i>[Contract C]</i> to 140 <br>
msg.sender: Value is 0x444444444444 <i>[Sending Account]</i>


To summarize :
<ul>
  <li>call and callcode both hide msg.sender</li>
  <li>call executes in called contracts context , callcode executes in the calling contracts context</li>
  <li>delegate call preseves the sender and executes in the calling contracts context</li>
</ul>

<br><br>
# Require vs assert

<ul>
<li>assert(false) compiles to 0xfe, which is an invalid opcode, using up all remaining gas, and reverting all changes.</li>
<li>require(false) compiles to 0xfd which is the REVERT opcode, meaning it will refund the remaining gas. </li>
</ul>


<br><br>
<hr>
# References
References: 
<ul>
<li><a href="https://ethereum.stackexchange.com/questions/3667/difference-between-call-callcode-and-delegatecallurl">Callcode vs delegate call</a></li>
<li><a href="https://ethereum.stackexchange.com/questions/15166/difference-between-require-and-assert-and-the-difference-between-revert-and-thro">Require vs assert</a></li>
</ul>
