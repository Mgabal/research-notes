# How DELEGATECALL Works at the Opcode Level — and Why It Breaks Proxies

*By Mahmoud Emad | Gabal — June 2026*

---

## What is DELEGATECALL?

When Contract A calls Contract B using a regular `CALL`, Contract B runs 
in its own context — its own storage, its own `msg.sender`, its own 
`address(this)`.

`DELEGATECALL` is different. Contract A borrows Contract B's **code** 
but runs it entirely inside Contract A's context. Contract B's code 
executes with Contract A's storage, Contract A's `msg.sender`, and 
Contract A's `address(this)`.

One sentence: **DELEGATECALL lets you run someone else's code in your house.**

---

## CALL vs DELEGATECALL Side by Side

| | CALL | DELEGATECALL |
|---|---|---|
| Code runs from | Contract B | Contract B |
| Storage modified | Contract B's | Contract A's |
| msg.sender inside B | Contract A | Original caller |
| address(this) inside B | Contract B | Contract A |
| msg.value inside B | Value sent | Value from original call |

This single difference — storage context — is what makes DELEGATECALL 
both powerful and dangerous.

---

## The Opcode Execution Trace

When the EVM encounters a DELEGATECALL instruction, here is exactly 
what happens?

Stack before DELEGATECALL:

[gas, addr, argsOffset, argsSize, retOffset, retSize]
EVM steps:

Pop all 6 arguments from stack
Load bytecode from address addr
Create a new execution frame
Set the new frame's storage context = CALLER's storage
Set the new frame's msg.sender = CALLER's msg.sender
Set the new frame's address(this) = CALLER's address
Execute the borrowed bytecode
Any SSTORE in the borrowed code writes to CALLER's storage slots
Push success (1) or failure (0) onto stack

I implemented this directly in my mini-evm project in Python:

```python
elif opcode == 0xF4:  # DELEGATECALL
    gas = self.pop()
    addr = self.pop()
    args_offset = self.pop()
    args_size = self.pop()
    ret_offset = self.pop()
    ret_size = self.pop()

    called_bytecode = self.contracts[addr]

    sub_evm = EVM(
        called_bytecode,
        caller=self.caller,   # original caller preserved
        origin=self.origin,
    )
    sub_evm.storage = self.storage  # CRITICAL: share storage
    sub_evm.run()
    self.storage = sub_evm.storage  # changes affect OUR storage
```

The key line: `sub_evm.storage = self.storage`. The called contract 
operates directly on the calling contract's storage. Every SSTORE 
in the borrowed code modifies the caller's state.

Full implementation: [github.com/Mgabal/mini-evm](https://github.com/Mgabal/mini-evm)

---

## Why Proxy Patterns Use DELEGATECALL

Ethereum smart contracts are immutable once deployed. If you find a bug, 
you cannot change the code at that address.

The proxy pattern solves this:

User → Proxy Contract (permanent address)

↓ DELEGATECALL

Logic Contract (replaceable)

The Proxy holds all the storage and ETH. The Logic contract holds the 
code. When you need to upgrade, you deploy a new Logic contract and 
point the Proxy at it. Users never change their address.

This works because DELEGATECALL runs the Logic contract's code inside 
the Proxy's storage context. From the user's perspective, nothing changes.

EIP-1967 standardizes the storage slots used to store the Logic 
contract address inside the Proxy, preventing accidental collisions.

---

## Why It's Dangerous — The Parity Hack

In 2017, a bug in Parity's multisig wallet library contract led to 
$150 million being permanently frozen.

The wallet used DELEGATECALL to execute library functions. A user 
discovered they could call the library's `initWallet()` function 
directly — because the library had never been initialized, they became 
its owner. They then called `kill()`, which executed `SELFDESTRUCT` 
inside the library's context.

But because of DELEGATECALL, `SELFDESTRUCT` ran in the context of every 
wallet that delegated to that library. Every wallet was destroyed. 
150 million dollars became permanently inaccessible.

The root cause: DELEGATECALL gives the called contract full power over 
the caller's storage and execution context. Any function the called 
contract can execute, it can execute on the caller's behalf.

---

## Storage Collision — The Silent Killer

Even without a hack, DELEGATECALL introduces storage collision risk.

Solidity assigns storage variables to slots sequentially starting at 
slot 0. If the Proxy and the Logic contract both declare variables, 
they occupy the same storage slots.

```solidity
// Proxy Contract
contract Proxy {
    address public implementation;  // slot 0
    address public owner;           // slot 1
}

// Logic Contract  
contract Logic {
    address public owner;    // slot 0 ← COLLISION with Proxy.implementation
    uint256 public balance;  // slot 1 ← COLLISION with Proxy.owner
}
```

When Logic writes to `owner` (slot 0), it overwrites 
`Proxy.implementation`. The proxy now points to a garbage address. 
Every subsequent call fails.

**This is why EIP-1967 exists.** It stores the implementation address 
at a pseudo-random slot derived from a hash:

```solidity
bytes32 private constant IMPLEMENTATION_SLOT = 
    bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1);
```

A keccak256-derived slot is astronomically unlikely to collide with 
any sequential storage variable.

---

## How to Audit a Proxy Contract

When reviewing a proxy-based protocol, check these in order:

**1. Storage layout compatibility**
Verify that the Proxy and Logic contracts do not declare overlapping 
variables in the same slots. Use `forge inspect ContractName storage` 
to view the full storage layout.

**2. Initialization protection**
Logic contracts must have an `initializer` modifier that prevents 
`initialize()` from being called twice. Uninitialized logic contracts 
are the Parity attack vector.

**3. Upgrade access control**
Who can call `upgradeTo(newImplementation)`? If it's not properly 
access-controlled, an attacker can replace the logic with a malicious 
contract and drain everything.

**4. SELFDESTRUCT in logic contracts**
If the logic contract contains SELFDESTRUCT, it can be triggered via 
DELEGATECALL and destroy the proxy's context. Search for `selfdestruct` 
and `delegatecall` in the same codebase.

**5. Function selector clashing**
The proxy's own functions (like `upgradeTo`) have 4-byte selectors. 
If a Logic contract function has the same selector, calls intended for 
the proxy get forwarded to the logic instead. Tools like 
`forge inspect` can detect this.

---

## Connecting to Nethermind

Nethermind's execution client processes every DELEGATECALL on the 
Ethereum network. At the client level, implementing DELEGATECALL 
correctly means preserving the caller's storage context, message 
sender, and address across the execution frame boundary — exactly 
what I implemented in mini-evm.

Understanding this opcode at the bytecode level is essential for 
execution client engineering, smart contract security research, and 
protocol design work. The difference between CALL and DELEGATECALL 
is one of the most consequential design decisions in the EVM 
specification.

---

## References

- [EIP-1967: Standard Proxy Storage Slots](https://eips.ethereum.org/EIPS/eip-1967)
- [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf)
- [Parity Multisig Hack Analysis](https:blogopenzeppelincomon-the-parity-wallet-multisig-hack)
- [mini-evm — My DELEGATECALL implementation](https://github.com/Mgabal/mini-evm)
- [evm.codes — DELEGATECALL reference](https://evm.codes/#f4)