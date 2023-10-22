# 工厂证明

- [📜 示例代码](./FactoryProofs.sol)
- [🐞 测试](../../test/FactoryProofs.t.sol)

许多协议都部署了多个可互操作的合约，但这些合约在发布时并不为人所知/尚未建立。众所周知的例子包括各种 [Uniswap](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol#L23) 版本和分叉，它们都为每个代币对部署了不同的池合约。在这些协议中，这些合约通常被赋予隐含的信任，使其表现良好（例如，swap 调用返回的金额就是你实际收到的金额）。

另一个更奇特/更极端的例子深藏在 0x 协议中，它有一个可插拔的“转换器”合约的概念，由调用者选择，实际上会被 [`delegatecall` 调用](https://github.com/0xProject/protocol/blob/development/contracts/zero-ex/contracts/src/features/TransformERC20Feature.sol#L272)。显然，`delegatecall` 调用任意合约的风险很高，因此该协议只允许调用由其控制的固定地址部署的合约。

通常会使用某种工厂模式来部署和验证链上合约的真实性。最简单的方法是始终通过工厂合约进行部署，并在其中存储有效部署地址的映射，以便日后查询。Uniswap V1 采用的就是这种方法。不过，这种方法在部署和验证时都会产生一些存储和间接 gas 开销。

## `CREATE2` 证明

从 Uniswap V2 开始，工厂使用 `CREATE2` 操作码来部署池合约，这意味着池地址可以是确定的，前提是每个池的创建 salt 都是唯一的。在 `CREATE2` 语义中，已部署合约的地址将由以下内容给出：

```soli
address(keccak256(abi.encodePacked(
    bytes1('\xff'),
    address(deployer),
    bytes32(salt),
    bytes32(keccak256(type(DEPLOYED_CONTRACT).creationCode))
)))
```

只要为 `DEPLOYED_CONTRACT` 实例提供（或可以获得）唯一的 salt，就可以简单地在链上执行此散列，以验证相关地址是否由`deployer`部署并可以信任--无需进行存储查找。

## `CREATE` 证明

但是，如果你不想使用工厂合约（`CREATE2` 只能由合约执行），或者你并不需要完全确定的地址，或者你正在使用传统协议/工厂，该怎么办？如果你知道部署者部署合约时的帐户 nonce，你仍然可以在链上证明合约是由某个地址部署的，而无需查找。

之所以能做到这一点，是因为即使是 `CREATE` 地址也有一定的确定性，只是用户对它的直接控制比 `CREATE2` 要少。在 `CREATE` 下，生成部署地址的唯一输入是 (1) 部署者的地址和 (2) 部署时部署者的账户 nonce，这两个输入都是简单的 RLP 编码和哈希值：

```solidity
// For how to implement rlpEncode, see: https://github.com/ethereum/wiki/wiki/RLP
address(keccak256(rlpEncode(deployer, deployerAccountNonce)))
```

对于 EOA（外部拥有的账户），账户 nonce 从 0 开始，每发送一笔挖矿的交易，账户 nonce 就递增一次。对于智能合约，账户 nonce 从 1 开始，每成功调用一次 `CREATE` 就递增一次。无论是哪种情况，你都可以在提供者/节点上使用 [`eth_getTransactionCount` JSONRPC 命令](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_gettransactioncount)来获取任意区块中某个地址的账户 nonce。

## 示例

这里提供的[示例代码](./FactoryProofs.sol)演示了如何在链上验证这两种部署。`verifyDeployedBy()` 用于验证部署者是否根据 `CREATE` 操作码语义部署了地址，而 `verifySaltedDeployedBy()` 用于验证部署者是否根据 `CREATE2` 语义部署了地址。