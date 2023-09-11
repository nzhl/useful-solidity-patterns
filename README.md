# Solidity 设计模式
---

这个仓库会持续收录一些在现实生产开发过程中有用或者比较巧妙的的 Solidity/EVM 设计模式。相关的内容会使用尽可能地用通俗易懂的方式来编写，以便降低阅读者的技术门槛。同时每个模式的文档都会附带简单, 可运行的代码示例和测试来更好地说明。

*仓库下所有的代码示例都只是教育用途, 某些地方为了更清晰地展示概念甚至放弃了最佳实践, 所以他们不应该在没有经过严格审计的情况下就直接应用到生产环境中.*


## [Solidity 设计模式](./patterns)
- [带选择器的 ABI-encoding 数据的解码](./patterns/abi-decode-with-selector/)
    - 用来解码函数调用以及错误信息的技术
- [错误处理进阶](./patterns/error-handling)
    - 对其他合约抛出的错误进行适当的处理以提升代码的健壮性
- [Assembly Tricks (Part 1)](./patterns/assembly-tricks-1)
    - Short, useful assembly tricks to save some gas and make up for solidity shortcomings.
- [Basic Proxies](./patterns/basic-proxies)
    - Contracts with upgradeable logic.
- [Big Data Storage (SSTORE2)](./patterns/big-data-storage)
    - Cost efficient on-chain storage of multi-word data accessible to contracts.
- [Commit + Reveal](./patterns/commit-reveal)
    - A two-step process for performing partially obscured on-chain actions that can't be front or back runned.
- [EIP712 Signed Messages](./patterns/eip712-signed-messages)
    - Human-readable off-chain messages that can be consumed on-chain.
- [ERC20 (In)Compatibility](./patterns/erc20-compatibility)
    - Working with both compliant and non-compliant (which are more common than you think) ERC20 tokens.
- [ERC20 (EIP-2612) Permit](./patterns/erc20-permit)
    - Perform an ERC20 approve and transfer in a *single* transaction.
- [`eth_call` Tricks](./patterns/eth_call-tricks)
    - Perform fast, complex queries of on-chain data and simulations with zero deployment cost using `eth_call`.
- [Explicit Storage Buckets](./patterns/explicit-storage-buckets)
    - Safer, guaranteed non-overlapping storage for upgradeable contracts.
- [Externally Owned Account Checks](./patterns/eoa-checks)
    - The consequences of interacting with contracts vs regular wallets, and how to identify them.
- [Factory Proofs](./patterns/factory-proofs)
    - Proving on-chain that a contract was deployed by a trusted deployer.
- [Initializing Upgradeable Contracts](./patterns/initializing-upgradeable-contracts)
    - Methods to safely and efficiently initialize state for proxy contracts.
- [Merkle Proofs](./patterns/merkle-proofs)
    - Storage efficient method of proving membership to a potentially large fixed set.
- [Multicall](./patterns/multicall)
    - Allow users to arbitrarily compose and perform multiple operations on your contract in a single transaction.
- [NFT Receive Hooks](./patterns/nft-receive-hooks)
    - Use ERC721/ERC1155 transfer callbacks to avoid having users set an allowance in advance.
- [Off-Chain Storage](./patterns/off-chain-storage)
    - Reduce gas costs tremendously by moving contract state off-chain.
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
