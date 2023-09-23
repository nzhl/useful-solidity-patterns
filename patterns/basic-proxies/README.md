# 代理模式基础

- [📜 示例代码](./ProxyWallet.sol)
- [🐞 测试](../../test/ProxyWallet.t.sol)

通常，智能合约的逻辑（字节码）一旦被部署就无法更改。对许多人来说，这是一件好事，因为它限制了开发者改变用户规则的能力。但更复杂、快速变化的协议通常需要更多的灵活性。代理模式允许开发者更改已经部署的合约的逻辑。总体来说，代理模式通常用于满足以下需求/场景：


- **迭代开发**: 通过更改已发布合约的逻辑，你可以向合约添加新的特性或功能，而无需迁移用户、转账授权或者是相关资产。
- **更便宜的部署**: 如果你发现自己多次部署相同的合约，使用代理模式可大幅度降低部署成本，因为代理合约代码简单体积小, 逻辑代码储存复用同一个份只需要部署一次。
- **合约体积限制**: 以太坊对部署的合约的代码大小有约24KB的上限，对于复杂的协议来说很容易超过这个限制。在代理架构中，逻辑可以分布在几个合约中，所以可以绕过单个合约可能会超过字节码大小限制的问题

## 工作原理


让我们深入了解一下实现基本代理所需要的步骤。代理模式利用了 EVM/Solidity 的两个特性：delegatecalls 和 fallback 函数。

### `delegatecall()`
在 Solidity 中，对另一个合约的常规函数调用被编译为使用 `address.call()` 或 `address.staticcall()` 语义。这两种调用类型都将在目标合约的上下文中执行函数，这意味着它只能访问自己的状态（地址，存储，余额等）。


但是，如果我们明确使用 `address.delegatecall()` 进行调用，它将使用 `address` 的代码逻辑执行调用，但是却在调用者的状态上下文中执行。这意味着任何存储读取和写入实际上都在调用者的存储中操作。目标合约继承了调用者的 `address(this)`、ETH 余额等上下文。因此，逻辑合约作出的任何外部调用也将像是来自调用者本身。就好像我们用目标合约的代码替换了我们自己(调用者)的代码。


### `Fallback` 函数
Solidity 允许你在合约上定义一个 `fallback()` 函数。对合约上未定义的函数的调用都会触发此函数。例如：

```solidity
function foo() external pure returns (string memory) {
    return 'in foo()';
}

fallback(bytes calldata callData) external returns (bytes memory rawResult) {
    revert('in fallback()');
}
```

如果你在合约上调用 `foo()`，它会返回字符串 `"in foo()"`。但假设你将这个合约的地址转换成一个 ERC20 接口，并尝试在它上面调用 `transferFrom()`。由于该函数在合约中未定义，fallback 将被触发，调用将以 `"in fallback()"` 的错误信息抛出错误。在这种情况下，`callData` 的值将是对 `transferFrom()` 进行 ABI 编码的结果（函数选择器+参数）。如果我们想从 fallback 返回任何内容，我们可以用 ABI 编码的返回数据填充 `rawResult` 并返回。


### 结合起来
使用代理模式，我们部署两个合约：一个代理合约和一个逻辑合约。逻辑合约本质上就是你通常编写的面向用户的业务合约，但用户 *不应该* 直接与它进行交互。相反，用户将与代理合约交互，代理合约将通过 使用 `delegatecall()` 将其调用传递给逻辑合约。

```
           call     ┌────────────────────┐              ┌────────────────────┐
       Proxy.foo()  │                    │              │                    │
User ───────────────►   Proxy Contract   │              │   Logic Contract   │
                    │                    │              │                    │
 ▲                  ├────────────────────┤ delegatecall └────────────────────┤
 │                  │                    ├──────────────►                    │
 │    foo() result  │     fallback()     │              │       foo()        │
 └──────────────────┤                    ◄──────────────┤                    │
                    ├────────────────────┐              └─────────┬──────────┘
                    │                    │                        │
                    │   storage/state    │                        │
                    │                    ├─ ── ── ── ── ── ── ── ─┘
                    └────────────────────┘       (shared state)
```

代理合约通常非常轻量, 因为它们唯一需要实现的逻辑就是将调用通过 fallback 转发给逻辑合约. 比如:

```solidity
contract Logic {
    bytes public message;

    function setMessage(bytes calldata message_) external {
        message = message_;
    }
}

contract Proxy {
    address immutable logic;

    constructor(address logic_) { logic = logic_; }

    fallback(bytes calldata callData)
        external
        payable
        returns (bytes memory resultData)
    {
        // forward the unhandled call to the logic contract, executing it in
        // our own state context.
        bool success;
        (success, resultData) = logic.delegatecall(callData);
        if (!success) {
            // bubble up the revert if the call failed.
            assembly { revert(add(resultData, 0x20), mload(resultData)) }
        }
        // Otherwise, the raw resultData will be returned.
    }
}
```

如果我们依托某个逻辑合约部署代理合约, 那么其实可以把代理合约当成逻辑的合约的一个实例, 具体表现如下:

```solidity
function test() external returns (string memory msg) {
    // Deploy a Logic contract
    Logic logic = new Logic();
    // Deploy a Proxy contract and set the implementation to the Logic instance.
    Proxy proxy = new Proxy(address(logic));
    // Treat the Proxy instance as a Logic contract.
    Logic proxifiedLogic = Logic(address(proxy));
    // call setMessage() on the Proxy instance, which will execute Logic's code
    // but store the message in the Proxy instance's storage context.
    proxifiedlogic.setMessage("i'm the proxy");
    // returns "i'm the proxy"
    return proxifiedLogic.message();
}
```

### 使得代理可升级

先前我们将逻辑合约地址存储为常量，并使用 `immutable` 修饰该字段，它将其值嵌入到代理合约的已部署字节码中，无法更改。但其实，我们可以将其设为常规的存储变量，这样就可以后续升级它。


在这样做时，我们需要 *极其小心地* 在代理合约中定义存储变量，因为编译器并不知道 `Proxy` 和 `Logic` 合约会共享相同的存储上下文，并可能将 `Logic` 中的存储变量分配到与 `Proxy` 中的位置重叠的位置（更多相关内容，请查看[显式存储桶模式](../explicit-storage-buckets/)）。存储逻辑合约地址建议的方法是遵循 [EIP-1822](https://eips.ethereum.org/EIPS/eip-1822) 或是 [EIP-1967](https://eips.ethereum.org/EIPS/eip-1967) 的规范。这两者都需要使用固定的、显式的（非编译器分配的）存储插槽来存储逻辑合约地址，我们可以用一些简单的 assembly 代码来获取它。所以如果要使得我们的 prox 代理能够适配 EIP-1967 ，我们会这样做：

```solidity
address immutable owner;
// explicit storage slot for logic contract address.
uint256 constant EIP1967_LOGIC_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

event Upgraded(address indexed logic); // required by EIP-1967

constructor(address logic) {
    owner = msg.sender;
    _setlogic(logic);
}

function upgrade(address logic) external {
    require(msg.sender == owner, 'only owner');
    _setlogic(logic);
}

fallback(bytes calldata callData)
    external
    payable
    returns (bytes memory resultData)
{
    address logic;
    assembly { logic := sload(EIP1967_LOGIC_SLOT) }
    (bool success, bytes memory resultData) = logic.delegatecall(callData);
    // ...same as before
}

function _setlogic(address logic) private {
    emit Upgraded(logic);
    assembly { sstore(EIP1967_LOGIC_SLOT, logic) }
}
```

Etherscan 能够识别 EIP-1967 模式的代理，将显示"read as proxy" 和 "write as proxy" 的选项卡，用于使用逻辑合约的 ABI 与你的代理合约交互。

![etherscan-proxy](./etherscan-proxy.png)

## 值得注意的问题
可以看到实现一个基本的代理模式并不难。虽然代理模式非常灵活和强大，但是许多黑客攻击都与他有关。所以为了安全地使用它，你需要始终注意它可能带来的许多风险。

### 保护逻辑合约

逻辑合约只是一个常规的合约，任何人都可以直接与之交互。通常这并不重要，因为直接使用逻辑合约时，任何状态更改都只会在逻辑合约的存储/账户上进行，而不会影响代理合约。然而，有一种状态改变可能对代理产生不利影响：`selfdestruct()`。如果有人能够直接在逻辑合约上调用 `selfdestruct()`，逻辑合约将发生自毁，任何将调用转发到它的代理合约将开始回滚（或者，更糟糕的是，静默成功），相关的资产将直接困在代理合约当中。这并不是空穴来风, 而是确有其事, 详见 [parity wallet hack](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7/) 


### 存储布局
在升级代理的逻辑合约时，必须极其谨慎，以确保存储布局仍然完全兼容旧的逻辑合约。改变存储变量声明的顺序，或者改变合约继承的顺序，可能会导致新的逻辑合约读写到和之前完含义完全不同的存储位置, 如下例:

```solidity
contract OldLogic {
    uint256 foo;
    bool canWithdraw;
    
    function incrementFoo() external { ++foo; }

    function withdraw() external {
        require(canWithdraw);
        payable(msg.sender).transfer(address(this).balance);
    }
}

contract NewLogic {
    // Delete `foo`
    bool canWithdraw;

    function withdraw() external {
        // canWithdraw will be non-zero (true) and this function can be called
        // if someone called incrementFoo() when oldlogic was in place!
        require(canWithdraw);
        payable(msg.sender).transfer(address(this).balance);
    }
}
```

为了避免上述情况, 最佳实践应该是只考虑新增变量而永远不在新的逻辑合约中删除, 插入已有的变量. 或者[压根不要依赖编译器为你分配卡槽位置](./../explicit-storage-buckets/)

### 重新初始化

合约的构造函数只在部署合约时被调用，并且总是在那个合约的状态上下文中执行。如果你需要在你的逻辑合约的构造函数中做一些变量的初始化工作，那么这些状态更改将不会反映在你的代理合约中。为了解决这个问题, 许多的代理合约实现中会将原本应该写在构造函数中的初始化逻辑移动到一个单独的初始化函数中, 这样代理合约就可以通过 `delegatecall` 的方式将这些状态的更新同步到自己身上

但是初始化函数也带来了一些风险, 比如需要注意确保这些初始化函数只能被调用一次, 否则的话有可能其他人可以通过重新初始化来替换掉你的关键配置以甚至给他们自己授予一些特殊的权限.

### 操作安全性

合约的升级机制可以并且应该被限制到管理员账户。管理员账户可以完全改变代理背后的逻辑合约，基于管理员账户进行卷款跑路非常简单, 因而管理员账户很容易成为黑客目标。通常，项目会将管理员账户放在多签名背后，以降低被攻击的影响, 但这只有在多签名签署者遵循安全实践时才能提供安全保障。作为另一条防线，升级功能可以被时间锁定和监控，以便用户和维护者有时间响应恶意的逻辑更改。


## 示例

[示例代码](./ProxyWallet.sol) 实现了一个代理化的钱包，其初始逻辑只能支持 ETH 但可以升级为也支持 ERC20 代币的逻辑合约。[测试](../../test/ProxyWallet.t.sol) 展示了如何将合约组合在一起

## 参考资料
- [OpenZeppelin 基本的合约代码](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol) and [一些需要注意的问题](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies) it. 
- OpenZeppelin 的许多 libraries 都有可升级版本, 以及关于如何使用他们的 [指南](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable) 
