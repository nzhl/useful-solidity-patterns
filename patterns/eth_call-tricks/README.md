# `eth_call` 技巧

- 代码实例:
    - [📜 swap forwarder](./swap-forwarder/)
    - [📜 swap forwarder with wallet unlock](./wallet-unlock-swap-forwarder/)

`eth_call` 是在 EVM 节点/提供者上常用的 JSONRPC 命令。当 dapp 想要评估智能合约上的只读（`view` 或 `pure`）函数的返回值时，底层的 Web3 库会在提供者（Alchemy、Infura、你自己的节点等）上执行一个 JSONRPC `eth_call` 命令。在另一端，处理请求的节点会执行函数调用（但不会真的上链这笔交易）并返回其结果。

但这个命令的功能远不止大多数人意识到的那么简单！在这里，我将展示由 MEV 机器人、聚合器、数据平台等使用的技巧。


## 调用非只读函数

令人惊讶的是，很少有 Web3 开发者意识到 `eth_call` 不仅仅适用于只读函数；它适用于任何函数！每一个主流的 Web3 库都有能力通过 `eth_call` 来评估任何非只读的合约函数：


```ts
// Assuming `doSomething()` is a non-view, non-pure function on a contract that
// returns a some value(s).

// making an eth_call in web3.js:
result = await contract.doSomething(...ARGS).call();

// making an eth_call in ethers.js:
result = await contract.callStatic.doSomething(...ARGS);
```

如果你想在将其提交到 mempool 之前先检查交易是否会成功, 或者如果你需要提前确认该调用的返回值, 那么这个技巧非常有用。

## 冒充其他账号或是篡改任意账户的 ETH 余额


`eth_call` 还允许你重写你发起调用的地址，以及将任何数量的以太坊设置到该调用地址上，而不管调用地址实际上拥有多少。

```ts
// overriding the caller and attaching arbitrary ETH to the call in web3.js:
result = await contract.doSomething(...ARGS).call({ from: SOMEONE_ELSE, value: ONE_ETHER });

// overriding the caller and attaching arbitrary ETH to the call in ethers.js:
result = await contract.callStatic.doSomething(...ARGS, { from: SOMEONE_ELSE, value: ONE_ETHER });
```
## Geth 重写 

现在我们来看一些真正有趣的东西！

Geth 节点（支持 Infura、Alchemy，并且是由 sidechains/L2s 主导的节点分支）支持可以传入 `eth_call` JSONRPC 命令的扩展参数。当调用被模拟时，这些参数允许你重写 EVM 状态的不同方面，包括：

- 任何地址的 ETH 余额。
- 任何地址的 Nonce。
- 任何地址的字节码。
- 任何地址中存储槽的值。


大多数 Web3 库不会方便地暴露出能够使用这些重写的 API，但你仍然可以通过一些底层的方法使用它们。


```ts
// geth's state overrides object
STATE_OVERRIDES = {
    [ADDRESS_TO_OVERRIDE]: {
        // Note: All fields are optional.
        balance: FAKE_BALANCE,
        nonce: FAKE_NONCE,
        code: FAKE_BYTECODE_HEX,
        stateDiff: { [SLOT_NUMBER_HEX]: FAKE_SLOT_VALUE_HEX, ...OTHER_SLOT_OVERRIDES },
    },
    ...OTHER_ADDRESS_OVERRIDES,
};
TX_OPTS = {
    to: TARGET_CONTRACT_ADDRESS,
    from: CALLER_ADDRESS,
    value: ETH_ATTACHED_HEX,
    gas: GAS_LIMIT_HEX,
    gasPrice: GAS_PRICE_HEX,
};

// making an eth_call with state overrides in web3.js:
// just need to do this bit once.
web3.eth.extend({ property: 'gethCall', methods: [{ name: 'eth_call', params: 3 }] });
// `result` will be ABI-encoded return value of the function call.
result = await web3.eth.gethCall(
    {
        data: contract.doSomething(...ARGS).encodeABI(),
        ...TX_OPTS,
    },
    'pending',
    STATE_OVERRIDES,
);

// making an eth_call with state overrides in ethers.js:
// `result` will be ABI-encoded return value of the function call.
result = await provider.send(
    'eth_call',
    [
        {
            ...contract.populateTransaction.doSomething(...ARGS),
            ...TX_OPTS,
        },
        'pending',
        STATE_OVERRIDES,
    ],
);


对于在 geth 下可用于 `eth_call` 的所有参数的完整说明，包括状态覆盖，请参阅 [它们的 JSONRPC 文档](https://geth.ethereum.org/docs/rpc/ns-eth) 。所有可能的覆盖都非常强大，但我认为最令人兴奋的是 `code` 覆盖，这正是我们接下来要探索的。
```


### 模拟部署合约

通过覆盖空地址（未部署的地址）的 `code`，对该地址进行的所有 `eth_call` 调用都会表现得好像该地址确实部署了一个合约。这也适用于你直接调用的合约。

但为什么你会想要调用一个实际上并不存在于链上的合约呢？通常，协议实际上会部署助手合约来支持其前端和后端所需的查询。通过使用 geth 的 `eth_call` ，您可以避免花费时间或金钱部署一个查询/助手合约来为您的链下服务！

另外，请记住 `eth_call` 只允许您执行一个函数调用，并且您在该调用中所做的任何事情都只是临时的（因为它没有实际上链）。所以，如果您想模拟复杂的交互，跨越多个、依赖的函数调用，在不同的合约之间，您可以使用自定义、假部署的合约作为中间人（我们将其称为 "Forwarder" 合约）来原子性地执行所有逻辑，并返回其调用结果。

#### 示例：模拟复杂的交换
让我们看一个 "Forwarder" 合约的示例，该合约输出 Sushiswap 和 Uniswap 之间复杂的 ETH -> USDC -> DAI 交换的结果（完整的工作示例可以在[此处](./swap-forwarder/)找到）：


```solidity
contract SwapForwarder {
    ...
    function swap() external payable returns (uint256 daiAmount) {
        IERC20[] memory path = new IERC20[](2);
        // WETH -> USDC leg on sushiswap.
        (path[0], path[1]) = (WETH, USDC);
        SUSHI_SWAP_ROUTER.swapExactETHForTokens{value: msg.value}(
            0, path, address(this), block.timestamp
        );
        // USDC -> DAI leg on uniswap (v2).
        USDC.approve(address(UNISWAP_ROUTER), type(uint256).max);
        (path[0], path[1]) = (USDC, DAI);
        UNISWAP_ROUTER.swapExactTokensForTokens(
            USDC.balanceOf(address(this)), 0, path, address(this), block.timestamp
        );
        return DAI.balanceOf(address(this));
    }
}
```

我们进一步编译该合约并通过如下方式调用它（基于 ethers）：
```ts
FORWARDER_ADDRESS = '0x123...'; // Some random address of your choosing.
forwarder = new ethers.Contract(FORWARDER_ADDRESS, FORWARDER_ABI, PROVIDER);
// Find out how much selling 1 ETH for USDC then DAI across sushi and uniswap gets us.
rawResult = await provider.send(
    'eth_call',
    [
        {
            ...(await forwarder.populateTransaction.swap()),
            value: ethers.utils.hexValue(ethers.constants.WeiPerEther),
        },
        'pending',
        { [forwarder.address]: { code: FORWARDER_DEPLOYED_BYTECODE_HEX } },
    ],
);
daiAmount = ethers.utils.defaultAbiCoder.decode(['uint256'], rawResult)[0];
```

#### 示例：解锁代币余额


你可以根据需要随时创建 ETH，方法是将一些 ETH 附加到函数调用中或为特定地址设置 `eth_call` 余额状态覆盖。但假设你想执行于之前 swap 相反的路径（DAI->USDC->ETH）。那么我们需要给 Forwarder 合约提供一些 DAI 代币。即使我们从一个确实拥有一些 DAI 的钱包地址发起对 Forwarder 合约的调用，这些 ERC20 代币也不能像 ETH 那样简单地附加到调用中。相反，该钱包地址首先需要将它们 `transfer()` 到Forwarder 合约，或者让 Forwarder 合约使用 `transferFrom()` 从钱包中提取它们，这在之前需要一个单独的 `approve()` 调用（也是从钱包地址发起的）。但请记住，我们只能在 `eth_call` 中直接调用一个函数。我们该如何解决这个问题呢？

可以说，获得 `eth_call` 中任意代币的最稳健方法是：


1. 找到一个拥有足够多代币的钱包。您可以简单地浏览 etherscan 的[最大持有者排名](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#balances)。
2. 用自定义合约覆盖该钱包地址上的 `code`，该合约会直接将资金转移到您的 Forwarder 合约中。


以下是我们用来替换钱包地址 `code` 的示例合约（完整的工作示例可以在[这里](./wallet-unlock-swap-forwarder)找到）:
```solidity
contract UnlockedWallet {
    function transferERC20(IERC20 token, address to, uint256 amount) external {
        token.transfer(to, amount);
    }
}
```

下面是基于它的修改版的 "forwarder" 合约:
```solidity
contract SwapForwarder {
    ...
    function swap(UnlockedWallet wallet, uint256 daiAmount) external payable returns (uint256 ethAmount) {
        // Pull DAI from the wallet.
        wallet.transferERC20(DAI, address(this), daiAmount);
        IERC20[] memory path = new IERC20[](2);
        // DAI -> USDC leg on uniswap (v2).
        DAI.approve(address(UNISWAP_ROUTER), type(uint256).max);
        (path[0], path[1]) = (DAI, USDC);
        UNISWAP_ROUTER.swapExactTokensForTokens(
            daiAmount, 0, path, address(this), block.timestamp
        );
        // USDC -> WETH leg on sushiswap.
        (path[0], path[1]) = (USDC, WETH);
        SUSHI_SWAP_ROUTER.swapExactTokensForTokens(
            USDC.balanceOf(address(this)), 0, path, address(this), block.timestamp
        );
        return WETH.balanceOf(address(this));
    }
}
```

然后这是我们需要如何调用 forwarder 合约的实例（基于 ethers）：
```ts
DAI_WALLET = '0xda1dadd1...'; // Address of a wallet with at least 100 DAI.
// Find out how much selling 100 DAI for USDC then ETH across uniswap and sushi gets us.
rawResult = await provider.send(
    'eth_call',
    [
        forwarder.populateTransaction.swap(DAI_WALLET, ethers.constants.WeiPerEther.mul(100)),
        'pending',
        {
            [FORWARDER_ADDRESS]: { code: FORWARDER_DEPLOYED_BYTECODE_HEX },
            [DAIL_WALLET]: { code: UNLOCKED_WALLET_DEPLOYED_BYTECODE_HEX },
        },    
    ]
);
ethAmount = ethers.utils.defaultAbiCoder.decode(['uint256'], rawResult)[0];
```

## 其他可能的应用

这些示例仅仅是探讨了 `eth_call` 覆盖的一些简单的用例。您还可以做的其他事情包括：


- 将批量链上查询组合成一个单一的 RPC 调用，提高响应速度并减少您的服务商费用。
- 模拟新的复杂部署、迁移、用户互动，甚至是漏洞利用。
- 编写链下辅助查询函数来增强已部署的合约。
- 覆盖状态，实现执行只有在某些条件下才会执行的代码，这些场景是 `eth_estimateGas` 是无法测试的，并且跟踪 gas 使用（使用 `gasleft()`）以找到一个异常的最大 gas 限制。


另一个代码覆盖的细微好处是，您的字节码不受 24KB 部署限制的约束 😉。


## 缺点

以这种方式使用 `eth_call` 时存在一些问题，特别是当您的互动变得更加复杂时：

- 状态在 `eth_call` 之间不会保存，所以您必须在一个单独的函数调用中完成所有的相互依赖的互动。有时，这可能需要一些不太直观的问题解决思路。
- `eth_call` 不会抛出任何在执行过程中应该抛出的事件。



## 对比本地 Forking
[Ganache](https://github.com/trufflesuite/ganache) 和 [Foundry](https://book.getfoundry.sh/) 支持创建一个针对当前实时网络情况的本地 VM 分叉。这通常是大多数人在针对已部署的生产协议进行测试时采取的方法，因为开发体验与使用真实网络相同。像 Foundry 这样的框架更加强大，因为您可以从测试合约内部覆盖 EVM 的几乎每个方面。

当您的模拟需要速度和保持最新状态时，本地分叉可能会遇到问题。本地分叉通过执行大量的状态读取 RPC 调用（例如，`eth_getStorage`，`eth_getCode`，`eth_getBalance` 等）来工作，下载被模拟的块的状态并缓存它们（在 Foundry 的情况下）。所以当第一次使用本地分叉时，这会导致显著的延迟（数秒），如果使用非常频繁的话可能会大幅增加你的服务提供商费用。相比之下，`eth_call` 是一个单一的 RPC 调用，不需要来回通信，并且通常在毫秒的时间顺序内完成。
