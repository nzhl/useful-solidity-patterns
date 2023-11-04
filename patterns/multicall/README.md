# Multicall
- [📜 实例代码](./TeamFarm.sol)
- [🐞 测试](../../test/TeamFarm.t.sol)

*⚠️ 请注意，这里指的不是 Maker 的 [`Multicall` utility contract](https://github.com/makerdao/multicall) 工具合约，它用于执行任意只读调用，通常在链下环境中使用*

对于与智能合约交互的用户来说，偶尔需要连续调用多个函数以完成单一目标的后续操作并不罕见。然而，大多数用户钱包（EOA）在一笔交易中只能进行单个顶级函数调用。许多协议只是简单地让用户在这些情况下进行多笔交易，或者为常见的组合操作创建专门的顶级包装函数（例如`wrapETHAndSwap()`）。

Multicall 模式通过向外暴露函数（`multicall()`）提供了一个简单而健壮的解决方案，该函数接受一个用户编码调用的数组并对其自身合约执行。这个函数遍历调用数组，并对每一个操作执行 `delegatecall()`。这允许用户组合自己的一系列操作，并在同一笔交易中顺序执行，而无需在协议中预先定义好某些操作组合。


![multicall 图示](./multicall-flow.png)

## 案例研究：一个共享的质押钱包

让我们用一个简单的合约 `TeamFarm` 来说明这种模式的有效性，它允许一群人将 ETH 和 ERC20 存入合约，并在一些 [ERC4626](https://ethereum.org/en/developers/docs/standards/tokens/erc-4626/) 金库中质押/取消质押这些资产。


合约有几个向外暴露的函数用于管理合约持有的资金和质押：


| 函数       | 描述       |
|-------------|---------|
| `deposit(token, tokenAmount)` | 将 ERC20 或 ETH 存入合约。 |
| `withdraw(token, tokenAmount, receiver)` | 从合约中取出 ERC20 或 ETH。 |
| `wrap(ethAmount)` | 将合约持有的 ETH 数量包装成 WETH。 |
| `unwrap(wethAmount)` | 将合约持有的 WETH 数量解包成 ETH。 |
| `stake(vault, assets)` | 将合约持有的代币数量（assets）质押到 ERC4626 金库，用资产创建份额。 |
| `unstake(vault, shares)` | 从 ERC4626 金库中取出合约持有的份额数量，将份额换回资产。 |

成员可以随时按任何顺序进行这些操作。假设一个成员想要存入 X 数量的 ETH 到合约，将其包装成 WETH，然后将该 WETH 质押到一个金库，他们需要进行以下一系列调用：


1. `deposit(token=0, tokenAmount=X)` (其中 `0` 地址指代原生 ETH )
2. `wrap(ethAmount=X)`
3. `stake(vault=WETH_VAULT_ADDRESS, assets=X)`

通常，每个调用都将是一笔独立的交易，产生额外的燃气费用，并且不能强有力地保证它们会被原子性地执行（全有或全无）。这对用户体验或可靠性来说都不是很好。


## 支持 Multicall

现在让我们引入一个 `multicall()` 函数，它接受一系列编码后的调用数据（`bytes`），我们将依次对其进行 delegatecall ：


```solidity
function multicall(bytes[] calldata calls) external payable {
    for (uint256 i = 0; i < calls.length; ++i) {
        (bool s,) = address(this).delegatecall(calls[i]);
        require(s, 'multicall call failed');
    }
}
```

由于 `delegatecall` 的语义，调用者的地址 (`msg.sender`)、发送的以太币值 (`msg.value`) 以及存储将被每个调用继承，而且由于 `delegatecall` 的目标是合约自身（`address(this)`），字节码也将会相同。这意味着每个调用都会如同调用 `multicall()` 的调用者直接调用那些函数一样被执行。这两个特性很重要，因为 `TeamFarm` 合约中几乎每个函数都对调用者有限制。

利用 multicall，前面例子中的 `deposit()`、`wrap()` 和 `stake()` 操作/调用可以通过将每个调用的编码数据传入 `multicall()` 来在同一个交易中执行。例如，在 ethers.js 中调用函数可能看起来像这样：

```ts
// Call `deposit()`, `wrap()`, and `stake()` all at once.
TEAM_FARM_CONTRACT.multicall([
    (await TEAM_FARM_CONTRACT.populateTransaction.deposit(ETH_AMOUNT)).data,
    (await TEAM_FARM_CONTRACT.populateTransaction.wrap(ETH_AMOUNT)).data,
    (await TEAM_FARM_CONTRACT.populateTransaction.stake(VAULT_ADDRESS, ETH_AMOUNT)).data,
], { value: ETH_AMOUNT });
```

## 对于 `payable` 函数需要注意的点

如果你的 multicall 中打算支持的函数包含有任何 `payable` 函数，那么 `multicall()` 函数本身也应该被声明为 payable。此外，如果某笔 `multicall()` 的交易调用中附带有 ETH，那么在这次交易中就不能将 `payable` 的函数与非 `payable` 的函数混合在改次交易中顺序调用。这是因为 delegatecall 的语义会继承顶层 `multicall()` 调用的 `msg.value`。而非 `payable` 函数会检查 `msg.value == 0`，所以如果 `multicall()` 调用时附带了 ETH 它们将会抛出异常。绕过这个问题的简单方法是将所有可以多次调用的函数添加为 `payable`，但你应该仔细评估这是否会引入任何安全隐患。



## 现实世界中的例子
- [Uniswap V3 的路由合约](https://github.com/Uniswap/v3-periphery/blob/main/contracts/SwapRouter.sol#L27)和 [position manager 合约](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol#L25)是 multicall 模式最常用的实现。
- UMA 协议中的 [`Multicaller`](https://github.com/UMAprotocol/protocol/blob/master/packages/core/contracts/common/implementation/MultiCaller.sol) 也提供类似的功能, 供 `HubPool` 和 `SpokePool` 合约调用。
- PartyDAO 的 [Party 协议](https://github.com/PartyDAO/party-protocol)在他们的[全局配置合约](https://github.com/PartyDAO/party-protocol/blob/main/contracts/globals/Globals.sol)上也使用了 multicall 的模式, 以便他们能够一次性更新多个配置参数。
