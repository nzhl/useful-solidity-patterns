# 显式存储桶

- [📜 示例代码](./ExplicitStorageBuckets.sol)
- [🐞 测试](../../test/ExplicitStorageBuckets.t.sol)

可升级合约（可替换字节码的合约）因其灵活性而成为一种极为常见的模式。要实现这种模式，通常需要使用一个透明"代理"合约，使用 `delegatecall` 操作码在代理的执行上下文中运行另一个合约的字节码，这样代理的状态（address, storage, balance 等）就被继承了。

> 文中出现的 address、storage、balance 等均为 Solidity 中的关键字，故没有进行翻译，下文相关部分同理。——译者注 

## 存储槽

为了更好地理解问题和解决方案，我们最好先了解一下 EVM 和 Solidity 编译器中的存储工作原理。

EVM 中的存储是以槽为基础的，每个槽位宽 32 字节。任何 256 位的数字都是有效的槽索引。存储空间的使用不要求连续，这意味着你可以随时读写任何槽。此外，如果相邻的 storage 变量都能容纳在 32 字节中，Solidity 编译器会尝试将它们打包到同一个槽中。

![storage slots](./storage-slots.png)

## 不要自掘坟墓

共享存储状态尤其不稳定，原因有以下几点：

- 升级代理的实现只会有效替换字节码，而不会替换存储状态。也就是说，如果实现合约的存储布局发生了变化（比如插入了一个新的 storage 变量），它可能会从无效的存储槽和位移中读/写。
- 编译器只会安全地堆叠来自相互继承的合约的存储变量。对于编译器来说，代理合约和它的实现合约实际上并不共享相同的状态。因此，代理使用的存储槽很可能会无意中与实现合约使用的存储槽重叠，从而导致数据损坏。

## 你说了算

编译器不可能知道你打算在代理模式中使用合约，但你知道。因此，与其依赖编译器分配存储槽，还不如手动定义"存储桶"，指向你选择的显式存储槽。由于 256 位整数空间非常大，为存储桶的起始槽选择唯一的哈希值绝不会与任何自动分配的槽重叠，也不会与任何其他存储桶重叠，如果你决定在共享执行上下文的多个合约中使用这种模式的话。

存储桶通过定义一个`结构体`类型来实现，该结构体类型可容纳通常在合约根级空间被定义为 storage 变量的所有字段。你甚至可以在存储桶结构中定义非原始和非连续的类型（如 mapping，array，struct），它们将继承存储桶的优点。此外，编译器仍会将相邻字段紧密打包在结构体中，因此你仍能从槽优化中受益。

要获得对存储桶的引用，需要使用一些低级汇编，将对该结构的引用手动指向存储槽。这样就可以通过熟悉的结构体语法访问 storage 变量了。

```solidity
contract StorageBucketExample {
    struct Storage {
        // Declare your private storage variables here rather than in the contract.
        uint256 foo;
    }

    constructor(uint256 foo_) {
        _getStorage().foo = foo_;
    }

    function foo() external view returns (uint256) {
        return _getStorage().foo;
    }

    function _getStorage() private pure returns (Storage storage stor) {
        assembly {
            // This value is just the hash of 'StorageBucketExample.Storage'
            stor.slot := 0x25440fdf23e3d55e3155d04a31ec5db1619e37c5a77b5eccf89b670f03ab1382
        }
    }
}
```

## 实际使用情况

- 据我所知，第一个在生产环境中使用这种模式的主要协议是 [0x V4 合约](https://github.com/0xProject/protocol/tree/development/contracts/zero-ex/contracts/src/storage)。
- 还有一种较新的可升级合约标准，称为”[钻石代理](https://eips.ethereum.org/EIPS/eip-2535)“，它也使用了存储桶。
- [标准代理存储槽](https://eips.ethereum.org/EIPS/eip-1967)标准是这种模式的精神先驱，因为它明确选择了一个存储槽来存储其实现地址。

## 示例

这里提供的[示例代码](https://github.com/nzhl/useful-solidity-patterns/blob/main/patterns/explicit-storage-buckets/ExplicitStorageBuckets.sol)使用代理实现了一个非常基础的可升级钱包。其目的是在创建后，只需要初始化一次，即可设置允许从钱包中提取 ETH 的地址列表。然而，我们给出了两个代理合约，其中一个合约（`UnsafeProxy`）很容易受到重新初始化攻击，因为它依赖于编译器分配的存储槽，而这些存储槽与执行合约的 storage 变量重叠。另一个安全版本的合约（`SafeProxy`）则不会受到攻击，因为它使用的是显式存储桶。[测试](https://github.com/nzhl/useful-solidity-patterns/blob/main/test/ExplicitStorageBuckets.t.sol)用例中演示了这种攻击的成功执行。
