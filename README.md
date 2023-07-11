# Yul - Solidity inline assembly
## Some example to understand what solidity does behind the scenes.

## Contents

-   [Syntax](#syntax)
-   [Types](#types)
-   [Basic Operations](#basic-operations)
-   [Functions](#functions)
-   [Storage Slots & Variables](#storage-slots--variables)
-   [Memory](#memory)
-   [Calls](#call)

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

