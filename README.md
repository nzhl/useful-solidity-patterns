# Solidity 设计模式
---

这个仓库会持续收录一些在现实生产开发过程中有用或者比较巧妙的的 Solidity/EVM 设计模式。相关的内容会使用尽可能地用通俗易懂的方式来编写，以便降低阅读者的技术门槛。同时每个模式的文档都会附带简单, 可运行的代码示例和测试来更好地说明。

*仓库下所有的代码示例都只是教育用途, 某些地方为了更清晰地展示概念甚至放弃了最佳实践, 所以他们不应该在没有经过严格审计的情况下就直接应用到生产环境中.*


## [Solidity 设计模式](./patterns)
- [带选择器的 ABI-encoding 数据的解码](./patterns/abi-decode-with-selector/)
    - 用来解码函数调用以及错误信息的技术
- [错误处理进阶](./patterns/error-handling)
    - 对其他合约抛出的错误进行适当的处理以提升代码的健壮性
- [汇编技巧-1](./patterns/assembly-tricks-1)
    - 一些精简, 有效的汇编技巧, 有助于节省 gas 同时绕开一些 Solidity 语言本身的限制
- [代理模式基础](./patterns/basic-proxies)
    - 可升级合约相关的逻辑
- [较大数据的存储 (SSTORE2)](./patterns/big-data-storage)
    - 一种更加节省gas的利用链上合约代码存储空间来储存任意数据（并可被合约读取）的方法，对较大的多词数据尤为有效
- [先提交 + 再揭晓](./patterns/commit-reveal)
    - 用一种“分两步走”的交易处理模式来产生“半模糊”的链上行为，用来防止抢跑交易和跟随交易。
- [EIP712链下消息签名](./patterns/eip712-signed-messages)
    - 可供人理解的JSON格式的信息在链下被签名，随后可在链上执行。
- [ERC20 的（不）兼容性](./patterns/erc20-compatibility)
    - 使用（比你想象中更加常见的）合规和不合规的 ERC20 代币。
- [ERC20 (EIP-2612) Permit](./patterns/erc20-permit)
    - Perform an ERC20 approve and transfer in a *single* transaction.
- [`eth_call` 技巧](./patterns/eth_call-tricks)
    - 使用 `eth_call` 执行快速且复杂的链上数据查询以及0成本的调用模拟
- [显式存储桶](./patterns/explicit-storage-buckets)
    - 为可升级合约提供更安全、有保证的非重叠存储。
- [Externally Owned Account Checks](./patterns/eoa-checks)
    - The consequences of interacting with contracts vs regular wallets, and how to identify them.
- [工厂证明](./patterns/factory-proofs)
    - 在链上证明合约是由可信任的部署人部署的。
- [Initializing Upgradeable Contracts](./patterns/initializing-upgradeable-contracts)
    - Methods to safely and efficiently initialize state for proxy contracts.
- [默克尔证明](./patterns/merkle-proofs)
    - 证明潜在大型固定集合成员资格的高效存储方法。
- [Multicall](./patterns/multicall)
    - Allow users to arbitrarily compose and perform multiple operations on your contract in a single transaction.
- [NFT 接收钩子](./patterns/nft-receive-hooks)
    - 使用 ERC721/ERC1155 转移回调以避免用户提前设置限额。
- [链下存储](./patterns/off-chain-storage)
    - 通过将合约状态移出链下，极大地降低 gas 成本。
- [OnlyDelegateCall / NoDelegateCall](./patterns/only-delegatecall-no-delegatecall/)
    - Restrict functions from being called from only within in a delegatecall context or not.
- [Packing Storage](./patterns/packing-storage)
    - Arranging your storage variables to minimize expensive storage access.
- [Permit2](./patterns/permit2)
    - Transfer tokens securely without a direct allowance, in a way that works for all (legacy and modern) ERC20s.
- [Read-Only Delegatecall](./patterns/readonly-delegatecall)
    - Execute arbitrary delegatecalls in your contract in a read-only manner, without side-effects.
- [Separate Allowance Targets](./patterns/separate-allowance-targets/)
    - Avoid having to migrate user allowances between upgrades with a dedicated approval contract.
- [Stack-Too-Deep Workarounds](./patterns/stack-too-deep/)
    - Clean solutions for getting around and avoiding stack-too-deep errors. So clean that you should do them regardless!
- 持续关注, 会有更多设计模式更新进来 😉

## Installing, Building, Testing

在开始之前确保你已经安装了最新版本的 [foundry](https://book.getfoundry.sh/getting-started/installation)

```bash
# Clone the repo
$> git clone git@github.com:dragonfly-xyz/useful-solidity-patterns.git
# Install foundry dependencies
$> forge install
# Run tests
$> forge test -vvv
# Run forked tests
$> forge test -vvv --fork-url $YOUR_NODE_RPC_URL -m testFork
```

## 感谢
本仓库是 [useful-solidity-patterns](https://github.com/dragonfly-xyz/useful-solidity-patterns/tree/main) 的中文译本, 非常感谢 [dragonfly_xyz](https://twitter.com/dragonfly_xyz) 为社区贡献了这么优质的内容.
