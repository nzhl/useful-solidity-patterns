# ERC20授权许可

- [📜 合约](./PermitSwap.sol)
- [🐞 测试](../../test/PermitSwap.t.sol)

接触过ERC20代币合约的用户普遍会感受到其一大痛点就是，合约要求用户单独发起一个 `ERC20.approve()` 链上交易（此函数会授权给一个地址一定的额度令其可以直接调取你的代币）方能允许某协议调动你的代币。这一步显得额外烦冗所以很多协议就默认简单化地请求用户给授权一个理论最大值(`2**256-1`)，这是一个基本上永远也用不完的额度，则用户无需再授权第二遍。然而，这个方法会将用户的全部数量的此类资产暴露在风险之下，若是此协议有漏洞则攻击者可能会盗走已被授权的全部资产。

[EIP-2612](https://eips.ethereum.org/EIPS/eip-2612)定义一种ERC20标准的拓展来允许用户对链下消息进行签名，则协议可立即使用此签名在链上获得对资产的授权使用额度。这意味着从用户那头只需要发起一个与协议交互的交易即可。得益于这种更高效的交互方式，协议可以每次都仅请求恰好所需的金额数量授权用来交互，安全上用户也会更加安心。

## 用户体验
这种模式在用户端看起来是怎样的？应用前端会让钱包app显示一个消息请求用户确认签名。这个消息包含了被授权方（这个协议），授权的数量，还有其他一些此模式要求的信息项。这一步是纯粹在链下发生的，不会发起链上交易。

![metamask permit sign prompt](./metamask-permit-sign-prompt.png)

用户签过字之后，紧接着会跳出又一个请求让用户确认与协议的链上交互，刚刚签字确认过的消息会被作为此交互的一个参数。在链上，协议用这个参数来授权给自己相应的额度，然后马上就兑现那个额度，调取用户的相应数量代币来进行被请求的链上行为。

## How it Works
First, for this pattern to work as described, the token must have EIP-2612 support (*but there exists [a universal workaround](#real-world-support) for legacy ERC20 tokens*).

Both on the frontend and smart contract side, [EIP-712 Signed Messages](../eip712-signed-messages) are employed to obtain and verify a human-readable signed message containing the permit details. Your contract will accept this signed message as an additional parameter and eventually passes into the `permit()` function of the token, granting itself an allowance, which it can immediately use.

![contract flow](./permit-flow.png)

### The Permit Message
The permit message the user signs has 5 fields:
| name | type | meaning |
|------|------|---------|
| `owner` | `address` | owner of the tokens (the user) |
| `spender` | `address` | who is being granted an allowance (the protocol) |
| `value` | `uint256` | how much `spender` can transfer from `owner` |
| `nonce` | `uint256` | an [on-chain incrementing value](#the-nonce) unique to `owner` that must match for this message to be valid (see below) |
|  `deadline` | `uint256` | timestamp after which this message expires |

All of these are fairly intuitive, with the exception perhaps of the `nonce`. This value corresponds to the current value given by the `nonces()` function on the token contract, and is specific to the owner/signer. A valid permit message must match the on-chain value when it is consumed by `permit()`, and that value will be incremented by one. Thus, after calling `permit()`, only permit messages with `nonce+1` are now valid, and so on. This prevents the permit message from being used again. This also means that if your protocol wants to queue up multiple transfers from the user using permits, you'll have to have the user sign successive nonce values, and redeem them in sequence.

## Contract Integration
The integration from the smart contract side is pretty simple, since the token contract will handle all the validation logic itself. Let's take a look at the [provided demo]((./PermitSwap.sol)), which takes the first token in `path` from the caller and swaps it for the last token in `path` on Uniswap V2 (V3 already supports permit 😏).

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
    // Consume the permit message. Note that we don't need to pass in `nonce`--
    // that value will be looked up on-chain.
    sellToken.permit(msg.sender, address(this), sellAmount, deadline, v, r, s);
    // Use our allowance to transfer `sellToken` to ourself.
    sellToken.transferFrom(msg.sender, address(this), sellAmount);
    // Grant the uniswap v2 router an allowance to spend our tokens.
    sellToken.approve(address(UNISWAP_V2_ROUTER), sellAmount);
    // Perform the swap.
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

## Real World Support
Support for permit is spotty but there are some major tokens that have either implement EIP2612 or something like it. Some examples include:
- [USDC](https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48#code)
- [AAVE](https://etherscan.io/token/0x7fc66500c84a76ad7e9c93437bfc5ac33e2ddae9#code)
- [UNI](https://etherscan.io/token/0x1f9840a85d5af5bf1d1762f925bdaddc4201f984#code)
- [DAI](https://etherscan.io/token/0x6b175474e89094c44da98b954eedeac495271d0f#code)
    - Slightly different [implementation]((https://github.com/makerdao/developerguides/blob/master/dai/how-to-use-permit-function/how-to-use-permit-function.md)) from EIP2612, but provides the same UX.

You might think legacy ERC20 tokens are out of luck, but there's actually a canonical contract that provides a workaround so users can enjoy a similar experience! [Permit Everywhere](https://github.com/merklejerk/permit-everywhere) is a simple and immutable contract that users can set traditional allowances on. Protocols can then accept permit-style messages and pass them into the Permit Everywhere contract to perform a transfer (skips the allowance step) on a user's behalf 🤯.

## Running the Demo (Forked)
The [`PermitSwap` demo contract](./PermitSwap.sol) relies on the mainnet Uniswap V2 Router contract, so to try it out you need to run the tests in fork mode:

```
forge test -vvv --match-path test/PermitSwap.t.sol --fork-url $YOUR_RPC_URL
```

## Resources
- [EIP-2612 Spec](https://eips.ethereum.org/EIPS/eip-2612)
- If you're planning on deploying a new ERC20 and want permit functionality, [OpenZeppelin has an implementation contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/draft-ERC20Permit.sol) you can inherit from.
