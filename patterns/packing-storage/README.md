# 打包存储

- [📜 示例代码](./PackedStoragePayouts.sol)
- [🐞 测试](../../test/PackedStoragePayouts.t.sol)

EVM 中的合约存储是围绕"槽"的概念构建的，每个槽位宽 32 字节，可以被任何 256 位的数字索引。在简单的情况下，编译器会根据你在合约中的声明，将存储变量分配到连续的槽中。

![slot storage](./slot-storage.png)

在单次操作中，读取和写入存储空间可能是合约定期进行的最昂贵的操作，单次读取成本高达 2,100 gas，单次写入成本高达 20,000 gas（在主网上）。非大型合约通常需要在单次交易中读取和写入多个存储变量，这意味着成本会迅速增加。

## 手动打包

减少访问多个存储变量带来的影响的一种方法是，尝试在一个槽中容纳多个变量。如果两个（或多个）存储变量的总大小不超过 32 字节，可以使用位操作将它们同时存储到一个槽中，这样就有可能将读写次数减少一半（或更多）。

```solidity
// Packed slot holding a uint64 and an address.
bytes32 _packedUint64Address;

function readPackedValues() public view
    returns (uint64 u64, address a160)
{
    // read the slot once
    bytes32 packed = _packedUint64Address;
    // uint64 is in the lower 64 bits
    u64 = uint64(packed & (2**64-1));
    // address is in next 160 bits
    a160 = address(uint160((packed >> 64) & (2**160-1)));
}

// Similar logic for writing...
```

但你不需要这样做！

## 自动打包
上述语法非常不优雅。幸运的是，Solidity 编译器会帮你完成这项工作，开箱即用。

编译器会尝试打包任何**相邻**的存储变量，只要它们能容纳在 32 字节内即可。如果它们不能放在同一个槽中，就会使用下一个槽，并从那里开始新的打包。由于这个过程，每个存储变量天然具有一个与之相关的"槽"（0 - 2^32-1）和字节"位移量"（0-31）属性。当你编写访问存储变量的 Solidity 代码时，编译器会生成执行位操作的代码，将变量与槽隔离开来，就像我们手动操作一样，但无需考虑它。

### 打包什么类型？

所有被声明为相邻且小于 32 字节的原始类型，即使是跨继承合约，也可以相互打包，包括：

- 整型:
    - `uint8`-`uint248`
    - `int8`-`int248`
- 固定宽度的字节:
  - `bytes1`-`bytes31`
- 布尔类型 `bool`
- 枚举类型 `enum`

声明为任何非原始类型（structs, arrays, mappings 等）的存储变量都会中断打包分配（S），并将该变量分配到一个新槽（S+1），即使当前槽尚未被完全使用。非原始类型不能与相邻变量一起打包，因此后面的变量也将从一个新槽（S+2） 开始。

某些类型也会将字段/值打包到自身中，但不会与相邻的变量一起打包。这包括 `struct`、固定宽度数组和短（长度小于 32）的 `bytes`/`string`。

以下面的合约为例：
```solidity
contract ContractA {
    bool foo;
}

contract ContractB is ContractA {
    struct MyStruct {
        uint32 bar;
        uint32 zing;
    }
    address who;
    MyStruct myStruct;
    uint16[3] things;
}
```

该合约将导致如下存储布局：

| 存储变量 | 槽 | 位移 | 长度 |
|------------------|------|--------|--------|
| `ContractA.foo`    | 0    | 0      | 1      |
| `ContractB.who`    | 0    | 1      | 20      |
| `ContractB.myStruct.bar` | 1 | 0 | 4 |
| `ContractB.myStruct.zing` | 1 | 4 | 4 |
| `ContractB.things[0]` | 2 | 0 | 2 |
| `ContractB.things[1]` | 2 | 2 | 2 |
| `ContractB.things[2]` | 2 | 4 | 2 |

### 检查分配的槽和位移量

大多数人不需要这样做，但你可以使用汇编程序找出与存储变量相关的槽和位移值，以便在代码中使用：
```solidity
uint64 _u64;
address _a160;

// Returns (0, 8)
function getAddressSlotAndOffset() external pure
    returns (uint256 slot, uint8 offset)
{
    assembly {
        slot := _a160.slot     // 0
        offset := _a160.offset // 8
    }
}
```

solc 编译器（以及 foundry、hardhat、truffle 等编译器）还可以[生成存储布局图](https://docs.soliditylang.org/en/v0.8.16/using-the-compiler.html#input-description)作为其构建工具的一部分，该图会输出合约中声明的每个存储变量的分配槽、位移量和长度。这对于在部署前获得正确的布局非常有用。

## 小窍门

### 一次查询
要想从存储打包中获得最大收益，一个好的经验法则是将经常一起读取或写入的变量紧挨着声明，这样编译器就会被提示尽量将它们放入同一个槽中。

### 使用较小的类型
选择的类型要与它们将持有的数值范围要能实际匹配。例如，如果你要跟踪的是 ETH、USDC 等的绝对真实数量，那么 `uint256` 类型可能会浪费你的比特，你可以使用 `uint128` 甚至 `uint96` 类型。对于时间戳，`uint40` 可以表示 34,000 多年（以秒为单位），这对于你的协议来说可能绰绰有余😉。但是，在这些情况下，向下类型转换时都要小心溢出。

### EIP-2929
[EIP-2929](https://eips.ethereum.org/EIPS/eip-2929) 引入了"冷"、"热"存储访问的概念。简而言之，首次访问存储槽会产生（至少） 2100 gas 的费用，但此后每次读取只需 100 gas。这就减少了重复访问存储空间的影响。因此，即使你不一定要在代码中的同一位置读取两个变量，只要它们是在同一事务中读取的，将它们打包在一起也是值得的，因为读取其中一个变量会减少对另一个变量的读取。请注意，这种行为只适用于那些已经实现 EIP-2929 的链。

## 工作示例

[提供的示例](./PackedStoragePayouts.sol)演示了按归属计划分配 ETH 的合约的两种实现（以说明 gas 差异）。这两种实现都跟踪通过 `vest()` 创建的每次派发的 4 个整数存储变量：

- `cliff`: 何时开始付款。
- `period`: 支付时间。
- `totalAmount`: 支付的 ETH 总额。
- `vestedAmount`: 已申领的 ETH 数量。

一个不成熟的实现（`NaivePayouts`）将这些变量都声明为 `uint256` 类型，这是新人开发者采用的典型方法。这意味着每次调用 `claim()` 都要读取 4 个存储槽。但是打包版本 (`PackedPayouts`) 会仔细选择每个变量的类型和语义，以确保它们的总和正好是 32 字节。这样，`claim()` 只需读取一个存储槽就能完成同样的工作。

你可以使用 `forge test --gas-report` 运行测试，以了解每种实现所使用的 gas。打包后的解决方案在 `claim()` 上gas 节约了 38% (21646 vs 33232)，在 `vest()` 上 gas 节约了 48% 😳 (47992 vs 93108)。还不错！
