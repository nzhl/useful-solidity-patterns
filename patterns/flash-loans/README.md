# 闪电贷

- [📜 示例代码](./FlashLoanPool.sol)
- [🐞 测试](../../test/FlashLoanPool.t.sol)

无论你认为它是好是坏，闪电贷俨然已经成为了现代去中心化金融系统架构中将会永远存在的一项机制。顾名思义，闪电贷允许用户来借出巨量价值的资产（有时量大到足以击垮一些协议的设计），仅在一个函数调用的周期之内完成“出借-使用-归还”系列操作，并且借贷费用是极小的，有时甚至没有费用。对于一些功能为资产托管的协议来说，闪电贷可以给其提供一种无风险的额外的收益。。。前提是它的实现机制是[安全的](#security-considerations) 🤞。

现在让我们通过尝试创建一个基本的闪电贷协议来阐述这一概念。

## 解析闪电贷

闪电贷的核心机制其实是相对简单的，流程如下：

1. 贷方合约将要出借的资产转移至一个用户所提供的借方合约处，完成出借。
2. 调用借方合约的处理函数来移交执行权。
    1. 借方合约的处理函数可任意去使用这些借来的资产，在这一步内完成使用。
3. 在借方合约的处理函数返回之后，贷方合约来检验所有出借的资产均已归还，并且还应有一些额外的部分作为收缴的费用。完成归还。


![闪电贷流程](./flash-loan-flow.drawio.svg)

整个借款并归还的流程都将在对贷出函数的调用之内完成。如果借款人无法在其借方合约的处理函数操作完成之后达到可以归还全部借款（加上费用）的状态，那么整个执行过程连带状态变量的更改都将会撤销，一切就像这笔借款请求从未发生过一样，所以没有资产会因此暴露于风险之中。正是这种无风险的特性，才使得闪电贷的借贷费用可以是很低的。

## 一个简易版闪电贷协议

Let's write a super simple ERC20 pool contract owned and funded by a single entity. Borrowers can come along and take a flash loan against the pool's tokens, earning a small fee along the way and increasing the total value of the pool. For additional simplicity, this contract will only support [compliant](../erc20-compatibility/) ERC20 tokens that don't take fees on transfer.

We're looking at the following minimal interfaces for this protocol:

```solidity
// Interface implemented by our protocol.
interface IFLashLoanPool {
    // Perform a flash loan.
    function flashLoan(
        // Token to borrow.
        IERC20 token,
        // How much to borrow.
        uint256 borrowAmount,
        // Address of the borrower (handler) contract.
        IBorrower borrower,
        // Arbitrary data to pass to borrower contract.
        bytes calldata data
    ) external;

    // Withdraw tokens held by this contract to the contract owner.
    function withdraw(IERC20 token, uint256 amount) external;
}

// Interface implemented by a flash loan borrower.
interface IBorrower {
    function onFlashLoan(
        // Who called `flashLoan()`.
        address operator,
        // Token borrowed.
        IERC20 token,
        // Amount of tokens borrowed.
        uint256 amount,
        // Extra tokens (on top of `amount`) to return as the loan fee.
        uint256 fee,
        // Arbitrary data passed into `flashLoan()`.
        bytes calldata data
    ) external;
}
```

Let's immediately flesh out `flashLoan()`, which is really all we need to have a functioning flash loan protocol. It needs to 1) track the token balances, 2) transfer tokens to the borrower, 3) hand over execution control to the borrower, then 4) verify all the assets were returned. We'll use the constant `FEE_BPS` to define the flash loan fee in BPS (e.g., `1% == 0.01e4`).

```solidity
function flashLoan(
    IERC20 token,
    uint256 borrowAmount,
    IBorrower borrower,
    bytes calldata data
) external {
    // Snapshot our token balance before the transfer.
    uint256 balanceBefore = token.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, 'too much');
    // Compute the fee, rounded up.
    uint256 fee = FEE_BPS * (borrowAmount + 1e4-1) / 1e4;
    // Transfer tokens to the borrower contract.
    token.transfer(address(borrower), borrowAmount);
    // Let the borrower do its thing.
    borrower.onFlashLoan(
        msg.sender,
        token,
        borrowAmount,
        fee,
        data
    );
    // Check that all the tokens were returned + fee.
    uint256 balanceAfter = token.balanceOf(address(this));
    require(balanceAfter >= balanceBefore + fee, 'not repaid');
}
```

The `withdraw()` function is trivial to implement so we'll omit it from this guide, but you can see the complete contract [here](./FlashLoanPool.sol).

## Security Considerations

Implementing flash loans might have seemed really simple but usually they're added on top of an existing, more complex product. For example, Aave, Dydx, and Uniswap all have flash loan capabilities added to their lending and exchange products. The transfer-and-call pattern used by flash loans creates a huge opportunity for [reentrancy](../reentrancy/) and price manipulation attacks when in the setting of even a small protocol.

For instance, let's say we took the natural progression of our toy example and allowed anyone to deposit assets, granting them shares that entitles them to a proportion of generated fees. Now we would have to wonder what could happen if the flash loan borrower re-deposited borrowed assets into the pool. Without proper safeguards, it's very possible that we could double count these assets and the borrower would be able to unfairly inflate the number of their own shares and then drain all the assets out of the pool after the flash loan operation!

Extreme care has to be taken any time you do any kind of arbitrary function callback, but especially if there's value associated with it.

## Test Demo: DEX Arbitrage Borrower

Check the [tests](../../test/FlashLoanPool.t.sol) for an illustration of how a user would use our flash loan feature. There you'll find a fun borrower contract designed to perform arbitrary swap operations across different DEXes to capture a zero-capital arbitrage opportunity, with profits split between the operator and fee.
