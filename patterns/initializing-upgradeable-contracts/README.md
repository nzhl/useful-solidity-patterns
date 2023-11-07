# 初始化可升级合约

- [📜 示例代码](./InitializedProxyWallet.sol)
- [🐞 测试](../../test/InitializedProxyWallet.t.sol)

当使用[代理](../basic-proxies/)模式时，通常你要部署一个空壳代理合约，这个合约做的事情就是通过使用 `delegatecall()` 简单地将*所有*调用命令转给另一个单独的逻辑合约。因为这个代理合约在理想情况下要被设计成一个泛泛的格式所以它不会有在逻辑合约里的各状态变量的定义，所以他也就无法独立完成这些状态变量在自身环境中的初始化。因此，开发者通常要明确地在逻辑合约内定义一个用来初始化的函数，然后令代理合约传递委托调用命令到逻辑合约，随后这个初始化函数就将在代理合约的环境内来被执行，进而完成那些状态变量在代理合约里的初始化。

![proxy with initializer diagram](./initializer.png)

## 示范
我们使用一个简单且通用的 `Proxy` 合约来开始对此模式做示范：

```solidity
contract Proxy {
    address public immutable LOGIC;

    constructor(address logic) { LOGIC = logic; }

    fallback(bytes calldata callData) external payable
        returns (bytes memory returnData)
    {
        // 通过委托调用将所有对proxy合约的调用转至对logic合约处执行
        returnData = _forwardCall(callData);
    }

    function _forwardCall(bytes memory callData)
        private returns (bytes memory returnData)
    {
        (bool s, bytes memory r) = LOGIC.delegatecall(callData);
        if (!s) assembly { revert(add(r, 0x20), mload(r)) }
        return r;
    }
}
```

假设我们想要代理一个基本的具有钱包功能的合约，这个合约能够接收 ETH 但是只有一个指定的钱包之主才能把钱转出来。那么在逻辑合约上，我们要定义一个 `initialize()` 函数来指定这个主人并且只能指定一次。

```solidity
contract WalletLogic {
    bool isInitialized
    address owner;

    // 指定主人且只能指定一次。
    function initialize(address owner_) external {
        require(!isInitialized, 'already initialized');
        isInitialized = true;
        owner = owner_;
    }

    // 把钱转走。
    function transferOut(address payable to, uint256 amount) external {
        require(msg.sender == owner, 'only owner');
        to.transfer(amount);
    }

    // 此合约可以接收ETH。
    receive() external payable {}
}
```

现在我们来创建这个钱包合约的实例：
1. 部署一个新的 `Proxy` 合约，传入已经部署过的 `WalletLogic` 逻辑合约的地址给代理合约的构造函数。
2. 对代理合约调用 `initialize()` 函数，这个调用会被转至 `WalletLogic` 逻辑合约处调用其带有执行逻辑的 `initialize()` 函数。
    1. 这样代理合约的环境中就会被指定一个 `owner` 状态变量并赋值。
    2. 同时 `isInitialized` 状态变量也会被赋值为 `true`，以后就都不能再重复执行 `initialize()` 了。

这是一种常见的初始化可升级合约的方式，并且也是开源项目 [Openzeppelin 库](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)里所采用的方式。这种方式通常在实践中没什么问题，但也会有一些小陷阱。

## 同时部署与初始化

一个很显然的问题就是代理合约要首先执行两个交互动作（先部署，然后调用 `initialize()`）才能让钱包开始正常工作。如果你从 EOA 钱包（非智能合约）处发起这两个交互，则需要两条交易，这就意味着别人有机会抢跑那条 `initialize()` 调用来设立一个别的 `owner`。这样的话就完球了！

解决这个的方法是，我们可以修改代理合约来在其构造函数中执行委托调用去 `initialize()`。但因为需要对其下达广泛形式的指令（因为你的代理合约不清楚其逻辑合约的 abi 是什么），所以我们实际上要传递进去直接*编码后的调令*，你可以直接使用 Web3 库（或 ethers 库，等等）来构造 `abi.encodeCall(WalletLogic.initialize, (owner))` 的值。现在只要代理合约被部署则同时也会被初始化！

```solidity
contract Proxy {
    constructor(Logic logic, bytes memory initCallData) {
        LOGIC = logic;
        // 通过委托调用自动执行 `initCallData`.
        _forwardCall(initCallData);
    }
    // ... 其余代码同上
}
```

## `isInitialized` 是必需的吗? 

回想一下 `WalletLogic` 逻辑合约使用了一个 `isInitialized` 状态变量来确保 `initialize()` 只会被执行一次。这里也有一些问题。

首先，此处没有什么机制来阻止其他人直接对 `WalletLogic` 逻辑合约去调用 `initialize()` （不通过 `Proxy` 代理合约），然后他成为了逻辑合约本身的拥有者。通常这个问题不大，因为这种方式给状态变量的赋值发生在逻辑合约环境内，而不会传递到代理合约处。但是，如果你的逻辑合约可以调用 `selfdestruct` 自毁或者自带其对别处的委托调用函数，那么就有可能有人来抢着初始化它，拿到所有权，然后自毁这个合约，这样也就毁了所有依赖于它的代理合约的功能。这正是之前发生过的 [Parity Wallet hack](https://blog.openzeppelin.com/on-the-parity-wallet-multisig-hack-405a8c12e8f7/)。

还有另外一个不算严重的问题就是此种方法需要花费 gas 来给 `isInitialized` 赋值，上限为 20k 个单位。我们的例子中还不太被这个问题所影响，因为 `isInitialized` 是紧接着一个 `address` 被声明的，这两个变量很舒服地被 [packs](../packing-storage/) 打包到同一个储存槽里去了，但是大多数项目所使用的标准化的实现方式 [OpenZeppelin implementation](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol#L66) 却是使用了补零填充来防止打包进同一个储存槽，所以这些合约就会花满这个额外的 20k 个单位的 gas 🙈。

那么有没有什么办法来既去掉 `isInitialized` 变量同时也保护逻辑合约不被直接初始化呢？

既然我们已经将对 `initialize()` 委托调用挪进了 `Proxy` 代理合约的构造函数之内，如果我们能确保 `initialize()` 函数*只能*从构造函数之内被调用，应该就不用担心它会被重复调用了。在 EVM 中，构造函数会返回此合约将被部署的地址中存在的字节码。所以，在构造函数内，`address(this)` 就代表了将被部署的地址，但此刻字节码还尚未存在在此地址中！所以，如果我们在构造函数未执行完毕时检查 `address(this).code.length`，即便是通过一个委托调用，我们仍会得到 `0` 值。所以现在我们可以升级 `initialize()` 函数逻辑来令他仅会在构造函数的内部来执行：

```solidity
contract WalletLogic {
    address owner;

    // 指定主人. 仅会在构造函数的内部来执行。
    function initialize(address owner_) external {
        require(address(this).code.length == 0, 'not in constructor');
        owner = owner_;
    }
    // ... 其余代码同上
}
```

现在 `Proxy` 代理合约的构造函数还是能委托调用 `initialize()`，但如果任何人在此代理合约部署之后还试图通过其来调用 `initialize()`，抑或是打算直接在 `WalletLogic` 合约上调用它，这个函数执行就会失败，因为 `address(this).code.length` 不为零，检查不会通过。并且，因为我们不需要使用任何状态变量来记录 `initialize()` 执行历史，那 20k 的 gas 也省了！实际上，我们所使用的这种检查只需要花 100 单位的 gas，相对于 20k 的标准化部署方案省了 200 倍的 gas 。泰酷辣！

## 实际中的用例
- [OpenZeppelin's 可升级合约](https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable)所用的都是传统的初始化模式。
- PartyDAO 的 [Party Protocol](https://github.com/PartyDAO/party-protocol) 代理合约则注重节省建立合约的成本。他们的逻辑合约定义了一个 [`onlyConstructor` modifier](https://github.com/PartyDAO/party-protocol/blob/main/contracts/utils/Implementation.sol#L24) 仅允许初始化在代理合约的部署过程中被执行。