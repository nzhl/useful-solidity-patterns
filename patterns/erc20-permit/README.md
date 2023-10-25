# ERC20授权许可

- [📜 合约](./PermitSwap.sol)
- [🐞 测试](../../test/PermitSwap.t.sol)

接触过ERC20代币合约的用户普遍会感受到其一大痛点就是，合约要求用户单独发起一个 `ERC20.approve()` 链上交易（此函数会授权给一个地址一定的额度令其可以直接调取你的代币）方能允许某协议调动你的代币。这一步显得额外烦冗所以很多协议就默认简单化地请求用户给授权一个理论最大值(`2**256-1`)，这是一个基本上永远也用不完的额度，则用户无需再授权第二遍。然而，这个方法会将用户的全部数量的此类资产暴露在风险之下，若是此协议有漏洞则攻击者可能会盗走已被授权的全部资产。

[EIP-2612](https://eips.ethereum.org/EIPS/eip-2612)定义一种ERC20标准的拓展来允许用户对链下消息进行签名，则协议可立即使用此签名在链上获得对资产的授权使用额度。这意味着从用户那头只需要发起一个与协议交互的交易即可。得益于这种更高效的交互方式，协议可以每次都仅请求恰好所需的金额数量授权用来交互，安全上用户也会更加安心。

## 用户体验
这种模式在用户端看起来是怎样的？应用前端会让钱包app显示一个消息请求用户确认签名。这个消息包含了被授权方（这个协议），授权的数量，还有其他一些此模式要求的信息项。这一步是纯粹在链下发生的，不会发起链上交易。

![metamask permit sign prompt](./metamask-permit-sign-prompt.png)

用户签过字之后，紧接着会跳出又一个请求让用户确认与协议的链上交互，刚刚签字确认过的消息会被作为此交互的一个参数。在链上，协议用这个参数来授权给自己相应的额度，然后马上就兑现那个额度，调取用户的相应数量代币来进行被请求的链上行为。

## 工作原理
首先，这个模式需要相应的代币合约支持EIP2612。(*但是老版ERC20合约也有[一种通用的解决方法](#real-world-support)*).

前端应用与后端合约都应用了[EIP-712消息签名](../eip712-signed-messages)来获取和验证一条人类可理解的，被加密签字过的消息，这个消息含有这条许可的细节。你的交互合约会接受这条被签署过的消息作为一个额外的参数传给代币合约的 `permit()` 函数，赋予交互合约以授权金额并且它可以立即使用。

![contract flow](./permit-flow.png)

### 授权许可消息
用户签署的许可消息含有5项内容:
| 变量名 | 变量类型 | 含义 |
|------|------|---------|
| `owner` | `address` | 代币的拥有者 (用户自身) |
| `spender` | `address` | 被赋予授权方 (交互的合约) |
| `value` | `uint256` | `spender` 可以使用 `owner` 多少数额的代币|
| `nonce` | `uint256` | 一个 `owner` 特有的[链上只用一次](#the-nonce)的数字，必须被代币合约验证一致这个消息方可有效(见下文) |
|  `deadline` | `uint256` | 消息有效性截止时间戳 |

上述诸项内容除了 `nonce` 以外其他的都很直观。这个数值是等同于调用代币合约的 `nonces()` 函数针对这个用户得到的值。一个有效的消息必须令此值与链上函数返回值一致方可继续运行 `permit()` 函数，每成功运行一次，这个值的链上返回就会加一，意味着此刻只有值为 `nonce+1` 的消息才被认为有效，如此方法继续下去。这样就防止了被签过的旧消息被重复使用。同时这也意味着如果你交互的协议想要以此从用户那里做多次的资产调动，使用此类许可的方式则需要用户签多次的消息并且按顺序依次执行。

## 与交互合约的整合
与交互合约的整合是简单的，因为代币合约承接了所有验证的逻辑。[这个例子]((./PermitSwap.sol))从 `path` 里获得代币合约地址，从用户那里拿到授权获得这些代币，然后将他们在Uniswap V2交易成另一种想要的代币（也在 `path` 里被指定）给用户。现在Uniswap V3已经直接支持授权许可 😏。

```solidity
function swapWithPermit(
    uint256 sellAmount,
    uint256 minBuyAmount,
    uint256 deadline,
    address[] calldata path,
    uint8 v,
    bytes32 r,
    bytes32 s
)
    external
    returns (uint256 boughtAmount)
{
    // 使用授权许可消息. 注意我们不用给permit函数额外传递`nonce`参数--
    // 因为函数会在链上调取它.
    sellToken.permit(msg.sender, address(this), sellAmount, deadline, v, r, s);
    // 使用授权将 `sellToken` 从用户处转至此合约处.
    sellToken.transferFrom(msg.sender, address(this), sellAmount);
    // 给uniswap v2 router授权来使用目前已经在合约处的这个代币.
    sellToken.approve(address(UNISWAP_V2_ROUTER), sellAmount);
    // 做交易
    uint256[] memory amounts = UNISWAP_V2_ROUTER.swapExactTokensForTokens(
        sellAmount,
        minBuyAmount,
        path,
        msg.sender,
        deadline
    );
    return amounts[amounts.length - 1];
}
```

## 现实中应用场景
对授权许可支持的应用还不算太多，但已有一些主流代币已经开始采用EIP2612这种，或与其相似的方式。一些例子如下：
- [USDC](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#code)
- [AAVE](https://etherscan.io/token/0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9#code)
- [UNI](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code)
- [DAI](https://etherscan.io/token/0x6b175474e89094c44da98b954eedeac495271d0f#code)
    - 一个EIP2612变体的 [实现方式]((https://github.com/makerdao/developerguides/blob/master/dai/how-to-use-permit-function/how-to-use-permit-function.md))，但从用户端来看体验是一样的.

你或许会认为传统老版的ERC20代币合约无法利用这种方式，但其实有一个典型的合约提供了一种解决方案来让用户获得相似的体验！[Permit Everywhere](https://github.com/merklejerk/permit-everywhere)是一个简单的不可改的合约，用户可对其授权。交互的协议可使用授权许可消息并将其传递给这个“Permit Everywhere”合约令其来执行一个代表用户的转账交易，同样跳过了链上的同意授权交易🤯。

## 运行样本合约 (用副本)
[`PermitSwap`样本合约](./PermitSwap.sol) 会在以太主网上与Uniswap V2 Router交互，所以想要跑测试的话先分叉一个副本在测试链上跑:

```
forge test -vvv --match-path test/PermitSwap.t.sol --fork-url $YOUR_RPC_URL
```

## 参考资料
- [EIP-2612 Spec](https://eips.ethereum.org/EIPS/eip-2612)
- 如果你想要部署一个ERC20代币并且想利用这个授权许可的模式，[OpenZeppelin有一个可将其实现的合约](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/draft-ERC20Permit.sol)你可以用来继承。
