# ERC20 的（不）兼容性

- [📜 示例代码](./ERC20Compatibility.sol)
- [🐞 测试](../../test/ERC20Compatibility.t.sol)

任何在以太坊上编写智能合约的人都不可避免地要与 ERC20 进行交互。从表面上看，该[标准](https://eips.ethereum.org/EIPS/eip-20)似乎足够简单明了，但该标准在历史上的实现方式不一致，可能会完全破坏协议的关键组件。这些不一致也不仅限于不常见的代币。举个真实的例子，试试在 [Uniswap V1 上交易 USDT](https://etherscan.io/address/0xc8313c965C47D1E0B5cDCD757B210356AD0e400C)😉。

本指南将介绍大多数使用任意 ERC20 代币的开发人员会遇到的两个问题，以及如何解决这些问题。

## 不一致的返回值行为

根据标准，所有修改 ERC20 状态的函数都应返回一个表示成功的 `bool` 值。因此，如果操作失败，函数要么返回 `false`，要么直接回退。通常情况下，ERC20 合约在操作失败时会选择（无论是否愿意）回退（例如，在尝试转出超额余额时），但也有少数合约在可能的情况下会选择返回 `false`。因此，检查调用的返回值也很重要。

当某些代币（[USDT](https://etherscan.io/address/0xdac17f958d2ee523a2206206994597c13d831ec7#code)、[BNB](https://etherscan.io/address/0xB8c77482e45F1F44dE1745F52C74426C631bDD52#code) 等）定义的 ERC20 函数在失败时进行回退，而在成功时未返回任何值时，情况就会变得特别糟糕。如果你通过通用且兼容的 ERC20 接口与这些合约进行交互，你的调用会在尝试解码 `bool` 返回值时发生回退，因为有时该值并不存在。

为了正确处理这些情况，我们需要使用[底层调用](https://docs.soliditylang.org/en/v0.8.17/units-and-global-variables.html#members-of-address-types)语义，这样返回值就不会被自动解码。只有当返回值存在时，我们才应尝试解码并检查它是否为 `true`。示例:

```solidity
// Attempt to call ERC20(token).transfer(address to, uint256 amount) returns (bool success)
// treating the return value as optional.
(bool success, bytes memory returnOrRevertData) =
    address(token).call(abi.encodeCall(IERC20.transfer, (to, amount)));
// Did the call revert?
require(success, 'transfer failed');
// The call did not revert. If we got enough return data to encode a bool, decode it.
if (returnOrRevertData.length >= 32) {
    // Ensure that the returned bool is true.
    require(abi.decode(returnOrRevertData, (bool)), 'transfer failed');
}
// Otherwise, we're gucci.
```

### 库

上述解决方案对于所有 ERC20 函数的变体都是一样的，而且现代 solidity 语法也足够清晰，因此自己实现 ERC20 令牌的通用处理并不难。但想要获得更加万无一失、开箱即用的解决方案，你需要导入 [OpenZeppelin 的 SafeERC20 库](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#SafeERC20)，该库将所有 ERC20 函数封装为"`safe`"变体，以便你更好的调用。

## 不一致的审批行为

在一些著名的 ERC20 中发现的另一个怪现象与设置津贴有关。在 ERC20 中，津贴是通过调用 `approve(spender, allowance)` 函数来设置的，该函数允许 `spender` 转移调用者代币的 `allowance`。通常情况下，调用 `approve()` 函数只会用新的津贴覆盖之前的值。但是，有些代币（USDT、KNC 等）只允许从 `0` 修改或者将 `allowance` 修改成 `0`。也就是说，如果你有津贴 `X`（其中 `X != 0`），要将其设置为 `Y`（其中`Y != 0`），你必须先将其设置为 `0` 😵 。这是一种预防措施，可减轻[此处](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit#heading=h.b32yfk54vyg9)概述的一种罕见的前置运行攻击。

因此，为了在更新津贴时获得普遍支持，在将津贴设置为非零值之前，还应（除了处理可选的返回值外）首先清除津贴：

```solidity
// Updating spender's allowance to newAllowance, compatible with tokens that require it
// to be reset first. Assume _safeApprove() is a wrapper to approve() that performs the
// optional call return value check as described earlier.
_safeApprove(token, spender, 0); // Reset to 0.
if (newAllowance != 0) {
    _safeApprove(token, spender, newAllowance); // Set to new value.
}
```

## 资源

本指南强调了在以太坊主网上使用任意 ERC20 时最常见的两个集成问题，但对于更奇特的应用，可能还有其他问题。如需更详尽的 ERC20 问题列表，请查看这个出色的 [Weird ERC20 代币资源库](https://github.com/d-xo/weird-erc20)。