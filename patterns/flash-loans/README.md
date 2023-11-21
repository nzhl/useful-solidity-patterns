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

我们来写一个超简易的ERC20资金池合约，仅由单一一方进行注资。借款者可以来对池子里的代币发起闪电贷，池子会收取一笔小小的费用进而持续增加池子的资金量。为了使其更加简化，这个合约只会支持对于转账操作不收取交易费的那些[标准合规的](../erc20-compatibility/)ERC20代币。

我们来看这个协议的最简化版的接口：

```solidity
// 我们的资金池的合约接口
interface IFLashLoanPool {
    // 闪电贷的函数
    function flashLoan(
        // 被出借的代币
        IERC20 token,
        // 出借数量
        uint256 borrowAmount,
        // 借方合约地址
        IBorrower borrower,
        // 要传递给借方合约的数据
        bytes calldata data
    ) external;

    // 将此合约持有的代币转给合约所有者的取钱函数
    function withdraw(IERC20 token, uint256 amount) external;
}

// 闪电贷借方合约的接口
interface IBorrower {
    function onFlashLoan(
        // 谁（地址）调用了 `flashLoan()`函数
        address operator,
        // 借到的代币
        IERC20 token,
        // 借到的数量
        uint256 amount,
        // 归还时额外要付的贷款费用
        uint256 fee,
        // 要传递回 `flashLoan()` 函数的数据
        bytes calldata data
    ) external;
}
```

让我们马上来具象化这个 `flashLoan()` 函数，基本上只要有了这个函数那么这个闪电贷合约就可以运作了。这个函数需要
1）记录代币的余额，
2）将代币转给借款方，
3）将代码执行控制转交给借款方让它去做它需要做的事情，
4）最后，检验所有的出借资产全部被归还。
我们使用常量 `FEE_BPS` 来定义贷款手续费的百分比基点（比如  `1% == 0.01e4` ）。

```solidity
function flashLoan(
    IERC20 token,
    uint256 borrowAmount,
    IBorrower borrower,
    bytes calldata data
) external {
    // 快照记录在转钱之前我们的代币余额
    uint256 balanceBefore = token.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, 'too much');
    // 计算贷款手续费，向上取整
    uint256 fee = (FEE_BPS * borrowAmount + 1e4-1) / 1e4;
    // 把代币转给借方合约
    token.transfer(address(borrower), borrowAmount);
    // 让借款人做他想做的操作
    borrower.onFlashLoan(
        msg.sender,
        token,
        borrowAmount,
        fee,
        data
    );
    // 检查所有资金都归还，并且费用也已付
    uint256 balanceAfter = token.balanceOf(address(this));
    require(balanceAfter >= balanceBefore + fee, 'not repaid');
}
```

在这里 `withdraw()` 取钱函数不重要所以我们先跳过它不讲，但是你可以查看完整的合约[在这](./FlashLoanPool.sol)。

## 安全考量

实现一个闪电贷逻辑看上去貌似比较简单，但是现实中通常这个逻辑是基于底层一个现成的并且更加复杂的产品基础之上。例如，Aave，Dydx和Uniswap都将闪电贷功能加到了他们的借贷和交易产品之中。这种闪电贷所使用的先转账再调用的行为模式会给[重入攻击](../reentrancy/)还有价格操纵这些攻击以很大的可乘之机。

例如，假设我们允许上面的这个简化版资金池顺其自然地升级它的特性，那么下一步的考量自然是允许其他人存入资产，给他们分以相应比例的费用收入作为其盈利。现在我们就需要思考一下假如闪电贷的借款人将借来的钱重新投入资金池，会发生什么？如果没有合适的安全机制来守卫，那么非常可能我们会重复计算这部分资产，这个借款人则以不公平的方式增加了自己对于池子的所占份额计数，然后他可以在闪电贷之后将池子掏空了！

如果在你的合约里存在允许任意的函数回调去执行，那么在设计上永远要谨慎再谨慎，尤其是当你的合约还涉及到真实有价值的资产的时候。

## 测试示例：闪电贷借款人对去中心化交易所进行套利交易

[这些测试](../../test/FlashLoanPool.t.sol)展示了用户怎样来使用我们的闪电贷功能。你会看到有这样一个有趣的借方合约，它是利用闪电贷的钱来跟不同的去中心化交易所进行反复的套利交易操作以获取空手套白狼的机会，操作的盈利自己留一些，剩下的用来交闪电贷的费用。
