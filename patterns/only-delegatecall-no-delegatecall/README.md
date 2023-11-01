# 仅允许委托调用 / 不允许委托调用

- [📜 示例代码](./DelegateCallModifiers.sol)
- [🐞 测试](../../test/DelegateCallModifiers.t.sol)

在[代理合约](../basic-proxies/)的架构中，一个简约化的代理合约利用它的fallback函数将所有对其调用命令转至另一个逻辑合约去执行。这个方法使用了低阶函数 `delegatecall()` 来令逻辑合约的代码在此代理合约的背景下执行。这意味着一些全局变量比如 `msg.sender`，`msg.value`，和 `address(this)`，还有所有的储存空间，都指的是代理合约的（发起 `delegatecall()` 的那个合约）。这就允许代理合约去跑其他合约的字节码，仿佛是它自己的一样。


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

## 逻辑合约也是独立的合约
在代理执行的架构中，用户应该直接与代理合约交互而不去直接接触逻辑合约。然而，逻辑合约本身也经常是完整有效带有各种可独立执行的函数的这样一种合约。所以容易被遗忘的一件事就是，通常情况下。逻辑合约无法阻止某用户直接对其发起交互。大多数时候这不算什么大事情，因为逻辑合约本身并不存在于你的产品体系之内。然而也存在某些情况下你会想要某个逻辑合约的函数不允许被直接调用，抑或是仅允许被直接调用而不允许委托调用。幸好，有一种既低价又简单的方式去做这两件事。

## `onlyDelegateCall` 仅允许委托调用
2017年[Parity多签钱包被黑](https://blog.openzeppelin.com/parity-wallet-hack-reloaded/)导致了约1.5亿美元价值的ETH被永久锁死在一众现已完全无用的智能钱包之中。每一个钱包个体都要做类似于代理执行的行为，将一些函数调用命令委托至一个被共享的库作为其逻辑合约来执行这些操作。这个逻辑合约含有一个初始化的函数，本意是来用作一个当某个钱包在建立的过程中可被其委托调用的函数，此函数可以初始化一些属于这个新钱包个体的状态变量值，并将其标记为已初始化所以不可再调用初始化。然而，开发人员没有考虑到有人会去*直接*在逻辑合约本身上调用这个函数。第一个做这件事的人可以令其自己成为这个逻辑合约的主人。这个动作本身，对于那些钱包个体来说，没什么大不了的，可惜的是，这个逻辑合约自己另带有一个 `kill()` 函数，若是被合约主人执行了这个函数则此逻辑合约将自毁。

一旦自毁， `address(this)` 合约的可执行字节码就会被抹去。如果 `kill()` 是由一个钱包个体通过委托调用来发起的，则这个钱包将被毁。但是如果此函数是直接在逻辑合约本体上发起调用的，那么它毁掉的就是逻辑合约。若是逻辑合约的可执行字节码已不存在，那么任何来自于钱包指向此逻辑合约地址的 `delegatecall()` 都将什么事都做不成。比如说，取钱功能就废了。

![self-destruct-a-la-parity](./parity-self-destruct.png)

The lesson from this is that if your logic contract can self-destruct OR if your logic contract can delegatecall to an arbitrary contract address (which could thereby self-destruct), you should strongly consider limiting the functions on that contract to only being called in a delegatecall context. Modern solidity (Parity did not have this luxury at the time) makes it quite easy to create a modifier, `onlyDelegateCall`, that restricts functions in exactly this way:

```solidity
abstract contract DelegateCallModifiers {
    // The true address of this contract. Where the bytecode lives.
    address immutable public DEPLOYED_ADDRESS;

    constructor() {
        // Store the deployed address of this contract in an immutable,
        // which maintains the same value across delegatecalls.
        DEPLOYED_ADDRESS = address(this);
    }

    // The address of the current executing contract MUST NOT match the
    // deployed address of this logic contract.
    modifier onlyDelegateCall() {
        require(address(this) != DEPLOYED_ADDRESS, 'must be a delegatecall');
        _;
    }
}
```

This example is of a base contract that you would inherit from in your logic contract and apply the modifier to risky functions. It works because:
- There is no way to delegatecall into a constructor so `address(this)` inside the constructor is always the deployed address of the currrent contract.
- Immutable types do not live in a contract's usual storage, but instead becomes a part of its deployed bytecode, so it will still be accessible even inside a delegatecall.
- Since `address(this)` is inherited from the contract that issued the `delegatecall()` and the immutable-stored `DEPLOYED_ADDRESS` stays with the bytecode being executed, inside of a delegatecall they will differ.

## `noDelegateCall`
Some protocols have tried to apply restrictive licenses that prohibit forking their code. But people have cunningly tried to circumvent these licenses by using `delegatecall()`, which could simply reuse the already deployed instance of existing contracts under the guise of another product. Both V1 and V2 of Uniswap have been forked so frequently to the point of becoming a meme but why don't we see the same trend with Uniswap V3? Uniswap V3 [went a step further](https://github.com/Uniswap/v3-core/pull/327#issuecomment-813462722) by outright preventing delegatecalls into many of their core contracts.

To create a `noDelegateCall` modifier, which is the inverse of the `onlyDelegateCall` modifier, we just flip the inequality. Now we *want* our execution context's address to match our logic contract's deployed address. If any other contract tries to delegatecall into our logic contract, `address(this)` will not match `DEPLOYED_ADDRESS`. Easy!

```solidity
// The address of the current executing contract MUST match the
// deployed address of this logic contract.
modifier noDelegateCall() {
    require(address(this) == DEPLOYED_ADDRESS, 'must not be delegatecall');
    _;
}
```

## The Example Code
[The example project](./DelegateCallModifiers.sol) that accompanies this guide implements both modifiers to be used inside a basic proxy architecture. We have two versions of a logic contract, `Logic` and `SafeLogic`, to demonstrate a potential use-case for each modifier. The `Logic` contract exposes two functions, each with vulnerabilities:

- `die()`, which is intended to self-destruct the *proxy* contract instance when called by the initialized owner. However, like in the Parity hack, the `Logic` contract can be initialized directly which can allow someone to call `die()` directly on it, bricking every proxy contract that uses it.
- `skim()`, which is a convenience function deisgned to allow anyone to take any ETH mistakenly sent to the logic contract. However, there is nothing stopping this function from being called through a proxy instance, which would mean all proxies that use this logic contract can have ETH taken out of them at any time.

The `SafeLogic` contract re-implements and corrects the vulnerable functions in `Logic` by applying an `onlyDelegateCall` modifier to `die()` and a `noDelegateCall` modifier to `skim()`. For examples on how to exploit the vulnerable functions, see the [tests](../../test/DelegateCallModifiers.t.sol).