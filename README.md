# Yul - Solidity inline assembly
## Some example to understand what solidity does behind the scenes.

## Contents

-   [Syntax](#syntax)
-   [Types](#types)
-   [Basic Operations](#basic-operations)
-   [Functions](#functions)
-   [Storage Slots & Variables](#storage-slots--variables)
-   [Memory](#memory)
-   [Others](#others)

## Description

Yul is an intermediate language that can be compiled to bytecode for different backends.

# Syntax

- To write Yul code in solidity, use `assembly` keyword.
```solidity
    contract C {
        function f() public {
            assembly {
                // code
            }
        }
    }
```

- To decleare variables, use the `let` keyword to declare variables.
```yul
    let zero := 0
```

# Types

- Yul has only 1 type: `bytes32`. This can hold any value. The compiler will automatically insert conversions as needed.
- For example, the following function will return `0x7461736962696900000000000000000000000000000000000000000000000000` which is the bytes32 representation of the `tasibii` string.
```solidity
    function f() public pure returns (bytes32 x) {
        assembly {
            x := "tasibii"
        }
    }
```

# Basic operations
### Yul has no overflow protection

|   Operators    |    Explaination    |
| :----------:   | :----------------: |
|    add(x, y)   |        a + b       |
|    sub(x, y)   |        a - b       |
|    mul(x, y)   |        a * b       |
|    div(x, y)   |        a / b       |
|    mod(x, y)   |        a % b       |

For multiple operations, the innermost operation is executed first:
```solidity
    let x := add(1, mul(2, 3)) // = add(1, 6) = 7
```

### Loops
```solidity
    assembly {
        for { let i := 0 } lt(i, 0x100) { i := add(i, 1) } {
            // do something
        }
    }
```

For loops can also be used as a replacement for while loops: Simply leave the initialization and post-iteration parts empty.
```solidity
    assembly {
        let i := 0
        for { } lt(i, 0x100) { } {     // while(i < 0x100)
            // do something
            i := add(i, 1)
        }
    }
```
The `break` and `continue` statements can be used in the body to exit the loop or skip to the post-part, respectively.

### If 
Yul has no boolean type. Instead, any value other than `0` is considered true.
```yul
    if iszero(0) {
        // if true, do something
    }
```

### Switch
Switch statements are similar to if statements, but they can only one case that is true and skip other case after one is true.

```solidity
    assembly {
        switch n
        case 0 {
            // if n == 0 do something
        }
        case 1 {
            // if n == 1 do something
        }
        default {
            // if neither case is true, do something
        }
    }
```

# Functions 
### Comparison operations
|     Operators     |         Explaination         |
| :--------------:  | :--------------------------: |
|     lt(x, y)      |   1 if x < y, 0 otherwise    |
|     gt(x, y)      |   1 if x > y, 0 otherwise    |
|     eq(x, y)      |   1 if x == y, 0 otherwise   |
|     iszero(x)     |   1 if x == 0, 0 otherwise   |

### Bitwise operations
|     Operators     |         Explaination         |
| :--------------:  | :--------------------------: |
|     and(x, y)     |   bitwise “and” of x and y   |
|     or(x, y)      |   bitwise “or” of x and y    |
|     xor(x, y)     |   bitwise “xor” of x and y   |
|     not(x)        |   bitwise “not” of x         |
|     byte(n, x)    |   nth byte of x              |
|     shl(x, y)     |    shift left y by x bits    |
|     shr(x, y)     |    shift right y by x bits   |

### Function declarations
```yul
    function functionName(param1, param2, ...) -> return1, return2, ... {
        // code
    }
    
    // example
    function power(base, exponent) -> result {
        switch exponent
        case 0 { result := 1 }
        case 1 { result := base }
        default {
            result := power(mul(base, base), div(exponent, 2))
            switch mod(exponent, 2)
                case 1 { result := mul(base, result) }
        }
    }
```

# Storage Slots & Variables

## Single variables being stored in one slot:
-   Storage slots are 256-bit words. To get the storage slot of a variable, use the `.slot` keyword.
-   To load a value from storage, use the `sload` keyword and pass in the storage slot as a parameter.
-   To store a value to storage, use the `sstore` keyword and pass in the storage slot and value as parameters.

```solidity
    uint256 x;
    
    function set(uint256 x_) public {
        assembly {
            sstore(x.slot, x_)
        }
    }
    
    function get() public view returns (uint256 x_) {
        assembly {
            x_ := sload(x.slot)
        }
    }
```

## Multiple variables being packed into one slot:
```solidity
    // Store position in slot
    // 0x 00 08 0010 000000000000000000000060 00000000000000000000000000000080   slot
    //       1   2             12                            16                  bytes 
    //       d   c              b                             a                  variables
    
    // Offset trick
    // 0x 00 08 0010 000000000000000000000060 00000000000000000000000000000080
    //          ------------------------ offset of d ------------------------- : 60 hex character => 30 bytes 
    //               --------------------- offset of c ----------------------- : 56 hex character => 28 bytes 
    //                                        --------- offset of b ---------- : 32 hex character => 16 bytes
    //                                                             offset of a :  0 hex character =>  0 bytes
      
    uint128 a;  // 16 byte
    uint96 b;   // 12 byte
    uint16 c;   //  2 byte
    uint8 d;    //  1 byte
    
    function set(uint16 _c) public {
        assembly{
            // Get the storage slot of the variable
            let wholeSlot := sload(c.slot)
    
            // Clear the variable's bits in the slot. Since it is a uint16, it is 2 bytes long.
            let cleared := and(wholeSlot, 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
            
            // The offset represented as bytes32
            // Shift the new value to the left by the offset of the variable multiplied by 8(1 byte = 8 bits)
            let shifted := shl(mul(c.offset, 8), _c)
            //                 ----- 224 -----
    
            // Combine the cleared slot and the shifted value
            let newValue := or(shifted, cleared)
    
            // Store the new value in the slot
            sstore(c.slot, newValue)
        }
    }
    
    function get() public view returns (uint16 c_) {
        assembly {
            // Get the storage slot of the variable
            let wholeSlot := sload(c.slot)
    
            // Shift the slot to the right by the offset of the variable
            let shifted := shr(mul(c.offset, 8), wholeSlot)
    
            // Mask the slot to get the value of the variable
            c_ := and(shifted, 0xffff)
        }
    }
```

## Storage Arrays
### Fixed Arrays
-   To get the bytes32 value at a specific index of a fixed array, use the `sload` keyword and pass in the storage `slot of the array + index` as parameters.
```solidity
    uint256[5] arr;
    
    function get(uint256 index) public view returns (uint256 value) {
        assembly {
            value := sload(add(arr.slot, index))
        }
    }
```
-   For arrays of variables smaller than 32 bytes, the compiler will pack the variables into a single slot when possible.
```solidity
    uint128[4] arr;
    
    function getFirstElement() public view returns (uint128 value) {
        bytes32 packed;
        assembly {
            // Get the first bytes32 of the array
            packed := sload(arr.slot)
            // Shift the bytes32 to the right by 16 bytes(128 bits) to get the value of the first variable
            value := shr(mul(16, 8), packed)
        }
    }
```

### Dynamic Arrays
- To get the bytes32 value at a specific index of a dynamic array, use the `sload` keyword and pass in the `keccak256 of the  storage slot of the array + index` as parameters.
- Can be written in the following ways:
```solidity
    uint256[] arr;
    function get(uint256 index) public view returns (uint256 value) {
        uint256 slot;
        assembly {
            slot := arr.slot
        }
        bytes32 location = keccak256(abi.encode(slot));
    
        assembly{
            value := sload(add(location, index))
        }
    }
    
    // or
    
    function get(uint256 index) public view returns (uint256 value) {
        assembly {
            mstore(0, arr.slot)
            value := sload(add(keccak256(0, 0x20), index))
        }
    }
```

## Mappings
### Simple Mappings 
Mappings behave similar to arrays, but it concatenates the key and the mapping's storage slot to get the location of the value.
```solidity
    mapping(uint256 => uint256) map;
    
    function get(uint256 key) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(key, uint256(slot)));
    
        assembly{
            value := sload(location)
        }
    }
    
    // or
    
    function get(uint256 key) public view returns (uint256 value) { 
        assembly {
            mstore(0, key)
            mstore(0x20, map.slot)
            value := sload(keccak256(0, 0x40))
        }
    }
```

### Nested Mappings
Nested mappings are similar, but use hashes of hashes to get the location of the value. The concatenation and the hashing is done from right to left.
```solidity
    mapping(uint256 => mapping(uint256 => uint256)) map;
    
    function get(uint256 key1, uint256 key2) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(key2, keccak256(abi.encode(key1, uint256(slot)))));
    
        assembly{
            value := sload(location)
        }
    }
    
    // or
    
    function get(uint256 key1, uint256 key2) public view returns (uint256 value) {
        assembly {
            mstore(0, key1)
            mstore(0x20, nestedMapping.slot)
            let hash := keccak256(0, 0x40)
            mstore(0, key2)
            mstore(0x20, hash)
            value := sload(keccak256(0, 0x40)
        }
    }
```

### Mapping of Arrays
```solidity
    mapping(address => uint256[]) map;
    
    function get(address key, uint256 index) public view returns (uint256 value) {
        bytes32 slot;
        assembly {
            slot := map.slot
        }
        bytes32 location = keccak256(abi.encode(keccak256(abi.encode(key, uint256(slot)))));
    
        assembly{
            value := sload(add(location, index))
        }
    }
    
    // or
    
    function get(address key, uint256 index) public view returns (uint256 value) {
        assembly {
            mstore(0, key)
            mstore(0x20, map.slot)
            let hash := keccak256(0, 0x40)
            mstore(0, hash)
            value := sload(add(keccak256(0, 0x20), index))
        }
    }
```

# Memory
### Memory is used for temporary storage of variables. It is cleared at the end of the function call.

### Memory is used in the following cases:

-   Return values to external calls
-   Set the function arguments for external calls
-   Get values from external calls
-   Revert with an error string
-   Log messages
-   Create other contracts
-   Use the keccak256 function

## Memory opcodes 
-   `mload(p)`: Retrieves 32 bytes from memory from slot p [p .. 0x20]
-   `mstore(p,v)`: Stores 32 bytes from v into memory slot p [p .. 0x20]
-   `mstore8(p,v)`: Like mstore, but only stores 1 byte
-   `msize`: Returns the largest accessed memory index in the current transaction

Example: Using `mstore` 7 into memory slot 0:
```solidity
    assembly {
        // empty memory looks like this:
        //  00   00   00   00  ...  00   00   00   00 
        // 0x00 0x01 0x02 0x03 ... 0x16 0x17 0x18 0x19
        // ---------------- 32 bytes -----------------
    
        mstore(0, 7) // ~ mstore(0, 0x00...07)
    
        // the memory now looks like this:
        //  00   00   00   00  ...  00   00   00   07 
        // 0x00 0x01 0x02 0x03 ... 0x16 0x17 0x18 0x19
    }
```

## How Solidity uses memory:

-   Solidity allocates slots [0x00-0x20], [0x20-0x40] for "scratch space" (first 32x2 bytes)
-   Solidity reserves slot [0x40-0x60] as the "free memory pointer" (the location of the next free memory slot)
-   Solidity keeps slot [0x60-0x80] empty (next 32 bytes)
-   The action begins at slot [0x80-...]

### To get the next free memory slot in Solidity, so you can use it knowing that it is empty, use the following code:

```solidity
    assembly {
        let freeMemoryPointer := mload(0x40)
    }
```

### Memory Structs
Adding structs to memory is just like adding their values 1 by 1.
```solidity
    struct S {
        uint256 a;
        uint256 b;
    }
    
    function f() external {
        bytes32 freeMemoryPointer;
    
        S memory s = S(a: 1, b: 2);
    
        assembly {
            // free memory pointer is now 0x80 + 32 bytes * 2 = 0xc0
            freeMemoryPointer := mload(0x40)
    
        mload(0x80) // returns a (1)
        mload(0xa0) // returns b (2)
        }
    }
```


### Memory Fixed Arrays
Fixed arrays work just like structs.
```solidity
    function f() external {
        uint256[2] memory arr = [1, 2];
    
        assembly {
            mload(0x80) // returns 0x0000...000001 (32 bytes)
            mload(0xa0) // returns 0x0000...000002 (32 bytes)
        }
    }
```

### Memory Dynamic Arrays
With dymanic arrays the first slot is used to store length of arrays.
```solidity
    function f(uint256[] memory arr) external {
        bytes32 location;
        bytes32 length;
    
        assembley {
            // the location will be the first free memory pointer: 0x80
            location := arr
            // the length will be the first memory slot of the array: 0x80
            length := arr.length
    
            mload(add(location, 0x20)) // returns the first element of the array
            mload(add(location, 0x40)) // returns the second element of the array
        }
    }
```

### abi.encode
The operation abi.encode will first push the bytes length of the arguments onto memory and then the arguments. If any argument is smaller than 32 bytes, it will be padded to 32 bytes.
```solidity
    function f() external {
        abi.encode(uint256(1), uint256(2));

        assembly {
            mload(0x80) // returns 0x0000...000040 (the bytes length of the arguments: 64)
            mload(0xa0) // returns 0x0000...000001 (32 bytes)
            mload(0xc0) // returns 0x0000...000002 (32 bytes)
        }
    }
```

### abi.encodePacked
If any argument is smaller than 32 bytes, abi.encodePacked will not add padding to the arguments.
```solidity
    function f() external {
        abi.encode(uint256(1), uint128(2));

        assembly {
            mload(0x80) // returns 0x0000...000030 (the bytes length of the arguments: 48)
            mload(0xa0) // returns 0x0000...000001 (32 bytes)
            mload(0xc0) // returns 0x00...0002 (16 bytes)
        }
    }
```

# Others
## return
The `return(a,b)` will take the data from memory, from slot a to slot b. This allows you to return data that is bigger than 32 bytes.
```solidity
    function f() external returns (uint256, uint256) {
        assembly {
            // store 1 and 2 in memory slots 0x80 and 0xa0
            mstore(0x80, 1)
            mstore(0xa0, 2)
            // return the data from slot 0x80 to slot 0xc0
            return(0x80, 0xc0)
        }
    }
```
If the return data is smaller than 32 bytes, it will not be padded to 32 bytes, so when the actual returned value is smaller than the value the client expects from the function statement, the client will not be able to decode the data. But if the return data is bigger than expected, it will just read the first x bytes it expects and will be able to decode the data.

## revert
The args of `revert(a,b)` are the same as `return(a,b)`, in the sense that it will also return the data from memory, from slot a to slot b. The difference is that `revert` will stop the execution of the function(it will not revert the whole transaction and the blockchain state as Solidity does).

```solidity
    assembly {
        if iszero(ez(caller(), 0xB0B)) {
            // This is the code used most of the time, just to stop the execution
            revert(0, 0)
        }
    }
```

## keccak256
In yul, the `keccak256(s,l)` will take the data to be hashed from memory, from slot s to slot s + l.

```solidity
    function f() external {
        assembly {
            // store 1 and 2 in memory slots 0x80 and 0xa0
            mstore(0x80, 1)
            mstore(0xa0, 2)

            // hash the data from slot 0x80 to slot 0xc0(0x80 + 0x40) and store it in slot 0xc0
            mstore(0xc0, keccak256(0x80, 0x40))
        }
    }
```

## Events
The Yul keywords for emitting events are:

-   `log0(p, s)` - emits an event with no topics and data of size `s` starting at memory slot `p`
-   `log1(p, s, t1)` - emits an event with one topic `t1` and data of size `s` starting at memory slot `p`
-   `log2(p, s, t1, t2)` - emits an event with two topics `t1`, `t2` and data of size `s` starting at memory slot `p`
-   `log3(p, s, t1, t2, t3)` - emits an event with three topics `t1`, `t2`, `t3` and data of size `s` starting at memory slot `p`
-   `log4(p, s, t1, t2, t3, t4)` - emits an event with four topics `t1`, `t2`, `t3`, `t4` and data of size `s` starting at memory slot `p`

The `t1` is the keccak256 hash of the event signature, and the `t2` is the first indexed argument of the event. The `t3` is the second indexed argument of the event, and so on.

```solidity
    event SomeLog(uint256 indexed a, uint256 indexed b, bool c);

    function f() external {
        assembly {
            // keccak256("SomeLog(uint256,uint256)")
            let signature := 0xc200138117cf199dd335a2c6079a6e1be01e6592b6a76d4b5fc31b169df819cc
            // store 1 in memory slot 0x80
            mstore(0x80, 1)
            // emit the event SomeLog(2, 3, true)
            log3(0x80, 0x20, signature, 2, 3)
        }
    }
```

## Check if a 32 bytes value is a valid Ethereum address
```yul
function isAddress(value) -> result {
    // 0xffffffffffffffffffffffffffffffffffffffff is used as a 32 bytes value by not, so it
    // will be padded to 0x000000000000000000000000ffffffffffffffffffffffffffffffffffffffff

    // the result of not will be
    // 0xffffffffffffffffffffffff0000000000000000000000000000000000000000

    // and a ethereum address is 20 bytes, so padded in 32 bytes looks like this:
    // 0x000000000000000000000000A4Ad17ef801Fa4bD44b758E5Ae8B2169f59B666F

    // so the result of AND will be 0 if the value is the length of an ethereum address
    if iszero(and(v, not(0xffffffffffffffffffffffffffffffffffffffff))) {
        result := 1
        leave
    }
}
```

## Checking for overflow in Yul
-   Addition: check if the sum of two values is less than either of the values.

```yul
function safeAdd(a, b) -> result {
    r := add(a, b)
    // OR is a smart way of the values is true, because if one of the values is 1, the result will be 1
    if or(lt(r, a), lt(r, b)) {
        revert(0,0)
    }
}
```

-   Subtraction: check if the minuend is lower than the subtrahend.

```yul
function safeSub(a, b) -> result {
    if lt(a, b) {
        revert(0,0)
    }
    result := sub(a, b)
}
```

-   Multiplication: if the product divided by the first value is not equal to the second value, then there is an overflow.

```yul
function safeMul(a, b) -> result {
    result := mul(a, b)

    if iszero(eq(div(result, a), b)) {
        revert(0,0)
    }
}
```

-   Division: check if the divisor is bigger than the dividend.

```yul
function safeDiv(a, b) -> result {
    if lt(b, a) {
        revert(0,0)
    }
    result := div(a, b)
}
```
