# 只读委托调用

- [📜 示例代码](./ReadOnlyDelegatecall.sol)
- [🐞 测试](../../test/ReadOnlyDelegatecall.t.sol)

委托调用可用于通过在状态上下文中执行不同的字节码/逻辑来扩展合约的功能。这也是绕过代码大小限制的有效方法。遗憾的是，`delegatecall()` 并没有像 `staticcall()` 和 `call()` 那样的"静态"（只读）版本，可以在状态发生变更时进行还原。因此，从本质上讲，委托调用可以自由地修改你的合约状态，或在其他地方执行变更状态的操作，同时冒充你的合约😱！因此，你绝对不会想在任意字节码中执行 `delegatecall()`，不是吗？

那么，如果你能保证执行的代码不会导致状态变更呢？在这种情况下，你的合约就可以愉快地在任意字节码中执行 `delegatecall()`，而不会对自身造成任何影响。这可能会解锁新的只读功能，使链上和链下集成变得更容易或更高效。

我们所要做的，就是想办法模拟一个或多个"静态" `delegatecall()` 🤗。

## 案例学习：无权限、任意、只读委托调用

我们的最终目标是在我们的合约上创建一个公共函数，让任何人都能传递合约的地址，并调用数据到 `delegatecall()` 中。这可用于在部署后"填补"合约上缺失或意外的 `view` 函数。它看起来类似于这样：

```solidity
function exec(address logic, bytes memory callData) external view;
```

它的返回值也应该与 `delegatecall()` 的返回值一模一样。但我们不会在假想的函数中声明这一点，因为我们实际上无法提前知道任意调用的返回数据是什么样的。但即使不声明它，我们也可以（并将会）使用一些底层汇编技巧来拿到返回数据，而不必了解其结构。因此，这样做是没有问题的。

## 方法 1：将 `delegatecall()` 封装到 `staticcall()` 中

如果在 `staticcall()` 内部发生的任何操作试图变更状态，`staticcall()` 都会还原。这种保护机制还延伸到了嵌套的 `call()` 中，甚至任何层级嵌套的 `delegatecall()` 中！因此，如果我们向实际执行 `delegatecall()` 的函数调用外部 `staticcall()`，我们就可以强制 `delegatecall()` 在其执行的任何代码试图变更状态时进行还原。

因此，我们需要定义 2 个 `external` 函数：`staticExec()`（无权限）和 `doDelegateCall()`（受限），这两个函数是这样工作的：

1. 用户在我们的合约上调用 `staticExec(logic,callData)`。
2. `staticExec() ` 执行 `staticcall()` 到 `this.doDelegateCall(logic, callData)`（在我们自己的合约上）。
3. `doDelegateCall()` 委托调用到 `logic` 中，调用带有 `callData` 的函数。
4. 我们将结果或者还原后的结果返回给用户。

让我们逐步编写实际执行委托调用的函数 `doDelegateCall()`。如果该函数还原了，我们就会向上传递（重新抛出）还原动作，但如果执行成功了，我们就会以 `bytes` 的形式返回结果。

请注意，尽管用户并不打算调用该函数，但它仍需声明为 `external`，以便 `staticExec()` 可以通过 `this` 语法实际调用它。此外，这个函数本身并没有任何 `staticcall()` 保护（这是下一步的内容），所以**重要的**是，除了合约本身，任何人都不能从合约外部调用这个函数！

```solidity
function doDelegateCall(address logic, bytes memory callData)
    external
    returns (bytes memory)
{
    require(msg.sender == address(this), 'only self');
    (bool success, bytes memory returnOrRevertData) = logic.delegatecall(callData);
    if (!success) {
        // Bubble up reverts.
        assembly { revert(add(returnOrRevertData, 0x20), mload(returnOrRevertData)) }
    }
    // Return successful return data as bytes.
    return returnOrRevertData;
}
```

接下来，我们定义用户将实际与之交互的函数 `staticExec()`。它会调用我们刚刚定义的 `doDelegateCall()` 函数，但使用的是 `staticcall` 上下文，然后返回原始的返回值 `bytes`，就像它自己返回的一样。我们会发现，仅调用 `this.doDelegateCall()` 将执行 `call()` 而不是 `staticcall()`，因为 `doDelegateCall()` 并未声明为 `view` 或 `pure`，而这并不是我们想要的。但是，如果我们将 `this` 传递给一个声明为 `view` 的 `doDelegateCall()` 的接口上（即 IReadOnlyDelegateCall），那么它将被 `staticcall()` 调用 🧙！

```solidity
interface IReadOnlyDelegateCall {
    function doDelegateCall(address logic, bytes memory callData)
        external view
        returns (bytes memory returnData);
}

...

function staticExec(address logic, bytes calldata callData)
    external view
{
    // Cast this to an IReadOnlyDelegateCall interface where doDelegateCall() is
    // defined as view. This will cause the compiler to generate a staticcall
    // to doDelegateCall(), preventing it from altering state.
    bytes memory returnData =
        IReadOnlyDelegateCall(address(this)).doDelegateCall(logic, callData);
    // Return the raw return data so it's as if the caller called the intended
    // function directly.
    assembly { return(add(returnData, 0x20), mload(returnData)) }
}
```

就是这样！这种方法非常优雅，因为它只依赖一个标准的、熟悉的 EVM 结构（`staticcall`）来执行不变性。

但对于某些合约来说，仅存在 `doDelegateCall()` 函数就很危险了，即使有 `msg.sender == this` 校验护航。如果你的合约可以任意调用用户传入的外部调用，或在其他地方执行委托调用，那么就有可能诱使合约在 `staticExec()` 的安全范围之外调用 `doDelegateCall()`。由于 `doDelegateCall()` 本身并不强制执行 `staticcall()` 上下文，因此任何未经授权的调用都可能导致实际的、持久的状态变更。对于这些情况，下一种方法提供了更加强有力的保障。

## 方法 2：委托调用后并还原

与其假设我们的 delegatecall 函数总是在 `staticcall` 上下文中被调用，我们可以强制执行，即使没有 `staticcall()`，其中的状态变更也不会持续。为此，我们只需在调用 `delegatecall()` 之后进行还原，这样就可以撤销当前执行上下文中发生的一切。我们在还原的数据中传输诸如 `delegatecall()` 的还原信息或者返回数据。这意味着函数将始终保持原样，撤销执行过程中发生的任何状态变更，而还原的内容则表征执行的结果。

因为这个函数不可能变更状态，所以我们再也不用担心它会被谁调用了。这就是我们新的、始终可还原的 `delegatecall` 函数，`doDelegateCallAndRevert()`：

```solidity
function doDelegateCallAndRevert(address logic, bytes calldata callData) external {
    (bool success, bytes memory returnOrRevertData) = logic.delegatecall(callData);
    // We revert with the abi-encoded success + returnOrRevertData values.
    bytes memory wrappedResult = abi.encode(success, returnOrRevertData);
    assembly { revert(add(wrappedResult, 0x20), mload(wrappedResult)) }
}
```

从技术上讲，该函数会执行并返回用户需要的所有信息，但代码中的还原操作并不直观。因此，我们仍将实现一个上层函数 `revertExec()`，该函数将调用 `doDelegateCallAndRevert()`，并对还原操作进行解码，像普通函数一样返回或抛出其结果。

```solidity
interface IReadOnlyDelegateCall {
    function doDelegateCallAndRevert(address logic, bytes memory callData)
        external view;
}

...

function revertExec(address logic, bytes calldata callData) external view {
    try IReadOnlyDelegateCall(address(this)).doDelegateCallAndRevert(logic, callData) {
        revert('expected revert'); // Should never happen.
    } catch (bytes memory revertData) {
        // Decode revert data.
        (bool success, bytes memory returnOrRevertData) =
            abi.decode(revertData, (bool, bytes));
        if (!success) {
            // Bubble up revert.
            assembly { revert(add(returnOrRevertData, 0x20), mload(returnOrRevertData)) }
        }
        // Bubble up the return data as if it's ours.
        assembly { return(add(returnOrRevertData, 0x20), mload(returnOrRevertData)) }
    }
}
```

与第一种方法相比，这种方法不太直观（谁会写一个总是还原的函数？），但其能提供强有力的安全保障。这两种方法的代码量都不大，所以如果你不确定要使用哪种方法，就用这种方法吧。😉

## 示例

[示例代码](./ReadOnlyDelegatecall.sol)是一个简单（无意义）的合约，它只有一个私有存储变量 `_foo`。由于 `_foo` 是 `private` 性质的，外部合约无法正常读取它的值。但由于它同时实现了 `staticExec()` 和 `revertExec()`，因此你可以使用其中任何一个来传递一个逻辑合约，该合约可以通过 `delegatecall()` 的黑魔法读取其存储槽。 在实际应用中，你还可以利用这个黑魔法对该状态执行额外的计算。[测试用例](.../../test/ReadOnlyDelegatecall.t.sol)中演示了如何使用它，以及如果逻辑函数在两种方法中都试图变更状态（显然都失败了），将会发生什么情况。


## 现实案例
- （[Gnosis](https://www.gnosis.io)）[Safe](https://safe.global) 使用委托调用和还原方法来[模拟](https://github.com/safe-global/safe-contracts/blob/v1.3.0-libs.0/contracts/handler/CompatibilityFallbackHandler.sol#L87)在安全合约上下文中执行交易的效果。
- Party 协议也使用了[委托调用和还原](https://github.com/PartyDAO/party-protocol/blob/e5be102b2cc2304768b21a3ce913cd28f2965089/contracts/utils/ReadOnlyDelegateCall.sol#L25)方法，以只读方式将[未处理的函数](https://github.com/PartyDAO/party-protocol/blob/e5be102b2cc2304768b21a3ce913cd28f2965089/contracts/party/PartyGovernance.sol#L325)转发到其 Party 合约的可升级组件中。
