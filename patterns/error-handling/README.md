# 错误处理

- [📜 代码示例](./PooledExecute.sol)
- [🐞 测试](../../test/PooledExecute.t.sol)

在与任何外部合约交互之前至少应该先考虑如何妥善处理他们可能抛出的错误。否则在极端情况下没有妥善处理的三方错误可能会导致我们的合约逻辑不能正确地执行。接下来我们将探索几种不同的错误处理方式。

## 抛出错误以及调用上下文

首先需要先了解一下抛出错误和 EVM 调用上下文之间的关系。

当一个合约调用另一个合约上的某个函数时（或者是该合约使用 `this.fn()` 的方式调用它自己时），将会创建并进入一个新的调用上下文。如果这时候发生了错误的话，在这个新的调用上下文中的代码执行将会立即中断，并且在这个调用上下文中发生的所有状态改变全部回滚（这就是为什么抛出错误的过程英文叫 revert，revert 的中文意思就是回滚）。如果此时在整个调用链中没有调用者拦截这次错误的话，则错误会层层冒泡最终导致整个交易失败。


![call-context-revert](solidity-call-reverts.png)

### 内部调用

值得一提的是如果我们调用 `internal`/`private` 函数或者调用 `public` 函数但不使用 `this` （也就是 `foo()` 而不是 `this.foo()`）的情况下实际上发生的是内部调用，此时在编译器会将调用编译成 `JUMP`，这时候的执行逻辑就好像所有的语句都被写在了同一个函数里面一样。换句话说此时的调用并不会创建调用上下文，所以如果这时候被调用函数抛出错误的话其实和调用者函数自身直接发生错误的效果是一样的，EVM 中没有办法拦截发生在函数自身的调用上下文中的错误，因此这种情况并不在我们的讨论范围之内。

## 两种不同的抛出错误的方式

通常来说合约抛出错误是通过内置函数 `require()`，`revert()` 或是 `revert` 关键字来完成的，这些情况下实际底层实现都是 `REVERT` 字节码。这也是比较推荐的在合约中抛出错误的方式。如上文所描述的，一旦抛出错误则当前调用上下文直接结束所有状态回滚并将控制权转移给上层调用者，同时返回错误信息（比如在 `require()` 的例子中返回的错误信息是通过参数来传递的，例如 `require(errorInfo)` )。

但是其实还有另一种更罕见的抛出错误的方式，底层使用的是 `INVALID` 字节码。一般很少有人会故意抛出这类错误，但有些老版本的编译器在处理整数溢出的时候会生成此类字节码，另外 EVM 自身的一些异常，比如 gas 消耗完了或者是调用栈太深的时候也会抛出这类错误。这两种错误最关键的区别在于，通过 `INVALID` 进行异常回滚会将所有的提供给该次调用的 gas 全部消耗掉，而 `REVERT` 则会返还剩余 gas。这个区别对于设计一个鲁棒性的合约来讲有重要的影响，具体细节会在[下文](#adding-more-resiliency)介绍。

## 错误处理

如前文所述, 当通过 Solidity 发起函数调用的时候，如果你不做特殊处理的话，如果有任何错误在该次调用中发生，会导致该错误继续向上冒泡进而导致你的调用逻辑也发生错误.

让我们来看看如何避免这种默认的行为，妥善处理抛出的错误而不是直接放弃😛。

### `try` + `catch`

Solidity `0.6.0` 中引入了和其他语言非常类似的 `try`/`catch` 语句来帮助我们处理错误。当然虽然看着像别的语言，Solidity 中的 `try`/`catch` 只能处理*单个函数调用*，该函数调用的语句必须紧接着 `try` 关键字，而在 `try` 的语句中描述的应该是调用成功后的后续逻辑，具体可以看下面的例子：

```solidity
try someContract.someFunction(arg1, arg2) returns (uint256 someResult) {
    // Call succeeded. Work on someResult.
} catch (bytes memory revertData) {
    // Call failed. Work on revertData.
}
// Rest of function...
```

如果你习惯使用 `require()` 或者 `revert()` 语法来抛出错误的话，可能会觉得有点奇怪因为 `catch()` 接收 `bytes`，如图中的 `revertData` 而不是一个 `string`，毕竟你传递给 `require()` 或者 `revert()` 的错误信息是一个 `string`。事实上，无论那种形式的错误信息最终都会转化为 `bytes`，并且这也是 `catch` 接受的唯一形式。在[下文](#inspecting-revert-data)我们将会继续解释其中的缘由。

### 底层调用

在 Solidity `0.6.0` 之前，使用底层调用（或者也可以使用汇编）是唯一可以捕获并处理错误的方式。底层调用主要是指 `call()`，`staticcall()`，或者 `delegatecall()`，他们都是 `address` 类型上的方法并且接受的是使用 ABI-encoding 编码过的数据（主要是包含函数的签名以及参数）作为参数。与直接抛出错误不同的是，这几个函数的返回值
都是 `(bool success, bytes returnOrRevertData)`，`returnOrRevertData` 的值在函数执行成功的时候返回函数的返回值否则返回的就是我们上面所说的错误信息 `bytes`。

```solidity
(bool success, bytes memory returnOrRevertData) = address(someContract).call(
    // Encode the call data (function on someContract to call + arguments)
    abi.encodeCall(someContract.someFunction, (arg1, arg2))
);
if (success) {
    // Process `returnOrRevertData` as encoded return data.
    uint256 someResult = abi.decode(returnOrRevertData, (uint256));
} else {
    // Process `returnOrRevertData` as encoded revert data.
}
```

显然这种方式比 `try`/`catch` 更啰嗦也更容易出错，因为后者可以在函数，调用参数以及返回值上给你更好的类型安全，而前者需要你自己编码函数签名参数并解码返回值。但是仍然有些场景下我们会需要使用这些底层调用, 具体在[下文](#adding-even-more-resiliency)会提到。

## 检查错误信息

既然我们现在已经理解了错误冒泡以及如何访问错误信息，那么要如何利用它呢？

你已经注意到了，所有例子中错误信息的类型都是 `bytes`。如果你习惯用 `require()` 或者 `revert()` 抛出错误，你可能会想为什么不直接返回类型 `string` 呢？原因是和返回数据一样，抛出的错误信息实际上可以是任何字节数组，使用关键字 `revert` 而不是 `revert() ` 函数允许你抛出一个自定义类型的的错误，当然错误也会使用 ABI-encoding 的
规则进行编码从而转化成 `bytes`。事实上，即使是你直接抛出一个 `string` 作为错误信息的情况下，它实际上也是先基于一个形如 `Error(string)` 的函数的形式进行 ABI-encoding 的编码成 `bytes` 然后再返回的，所以在这个例子中实际返回的是形如 `0x08c379a00000000000000000000000000000000000000000000000000000000000000020...` 的
形式。

```solidity
revert('hello')
// ^ is equivalent to:
error Error(string msg); // Declare custom error type
...
revert Error('hello'); // Throw custom error type
```

因此，假设我们想要在合约抛出字符串 `foo` 的时候进行捕获并处理，那么我们只需要比较实际返回的错误信息的哈希与签名为`Error(string)` 参数为 `foo` 的 ABI-encoding 编码结果之后的哈希就行了。之所以不直接比较两个返回值而是比较返回值的哈希是因为比较哈希通常比比较字节数组更快（我个人认为更准确的说法应该是在 Solidity 中比较字节数组一个比较方便的方法就是比较哈希，因为 Solidity 中不支持数组直接比较，另一种做法就是逐个字节进行比较，也非常麻烦），具体操作如下：

```solidity
try someContract.someFunction(arg1, arg2) returns (uint256 someResult) {
    ...
} catch (bytes memory revertData) {
    if (keccak256(revertData) == keccak256(abi.encodeWithSignature('Error(string)', ('foo')))) {
        // someContract.someFunction() failed with error string 'foo'
        ...
    }
}
```


如果抛出的错误信息涉及复杂类型，或者具有多个不同参数，你可以参考我们上一节提到的[带选择器的 ABI-encoding 数据的解码](../abi-decode-with-selector/)。

## 手动向上抛出错误

有时候可能会发现我们并不想处理某些错误，而是继续抛出希望上层调用者来处理，这时候初学者可能会尝试使用 `revert()` 来再次抛出错误：

```solidity
(bool success, bytes memory returnOrRevertData) = someContract.call(...);
if (!success) {
    // Try to bubble up the revert data to our caller by force-casting the
    // revert data to a string type because `revert()` only accepts a string.
    // This is wrong!!! 🪲
    revert(string(returnOrRevertData));
}
```

但是这其实忽略了一点，错误数据本身很大可能已经是一个 ABI-encoding 编码过的 `Error(string)` 类型，我们这里使用 `revert()` 再次抛出错误会导致实际的错误信息被 `Error(string)` 的形式编码两次，所以正确的方式应该是将错误数据原封不动的向上抛出, 这需要用到简单的汇编：

```solidity
(bool success, bytes memory returnOrRevertData) = someContract.call(...);
if (!success) {
    // Bubble up the revert data unmolested.
    assembly {
        revert(
            // Start of revert data bytes. The 0x20 offset is always the same.
            add(returnOrRevertData, 0x20),
            // Length of revert data.
            mload(returnOrRevertData)
        )
    }
}
```

## 增加更多的鲁棒性

如果你的代码调用了你没有仔细检查或者有可能遇到 `INVALID` 字节码的合约时，这时候建议给这次调用加上 gas 上限。否则的话，这次调用有可能会消耗掉所有剩余的 gas（译者注：这里说消耗完所有的应该不准确，因为事实上在 Tangerine Whistle fork 以后，发起调用默认情况下（最多）只能消耗调用者调用上下文中 63/64 的 gas）。注意这里说的 gas 限制限制的是*最大*数量的 gas 消耗而不是最小。同样在 `try`/`catch` 的例子中加上 gas 限制的代码如下：

```solidity
// Restrict a try/catch call to 500k max gas.
try someContract.someFunction{gas: 500e3}(arg1, arg2) returns (uint256 someResult) {
    ...
} catch {
    ...
}
// Restrict a low-level call to 500k max gas.
(bool success, bytes memory returnOrRevertData) = address(someContract).call{gas: 500e3}(
    abi.encodeCall(someContract.someFunction, (arg1, arg2))
);
...
```

## 增加甚至更多的鲁棒性

也有一些 `try`/`catch` 没办法处理的极端情况，这些情况下错误是由编译器生成的额外检查抛出的，他们本质属于*你的函数的一部分*而不是目标函数，所以错误会被直接抛出而没有办法被截获，具体而言：

- 你希望*如果调用目标地址没有代码则直接返回空值*而不是调用失败时，你的调用实际会抛出错误。
    - 这是因为编译器会生成在调用之前预先检查是否目标地址中含有代码的逻辑。
    - 被调用的合约不包含代码可能是因为它就没有被部署过或者已经自毁了 (还是可能它就是个 EOA 地址)。
- 当被调用的函数的返回值并不能被成功解码成我们预期的类型，也会抛出错误。
    - 比如说一个函数预期返回 `uint256` 但是实际上返回了个少于 32 字节的数据。
    - 这种情况有可能是因为合约本身就是个恶意合约也可能是合约没有正确实现约定的规范。

为了更优雅地处理这些极端情况，用回低级调用可能会更好，原因如下：
- 编译器不会自动生成预先检查目标合约是否含有代码的逻辑。
- 我们可以在使用 `abi.decode()` 解码返回数据之前手动检查其正确性从而避免出现类似使用 `try`/`catch` 方式抛出的错误。

```solidity
(bool success, bytes memory returnOrRevertData) = address(someContract).call(
    abi.encodeCall(someContract.someFunction, (arg1, arg2))
);
if (success) {
    if (returnOrRevertData.length >= 32) {
        // Successful and returned a decodable uint256.
        uint256 someResult = abi.decode(returnOrRevertData, (uint256));
        ...
        return;
    }
}
// Either the call failed or the return data was too short to hold a uint256.
...
```

调用一个没有代码的地址（EOA 或者还未部署的地址）的默认行为是成功以及空数据。所以在调用成功一定会返回数据的场景下，我们可以不需要检查我们是否调用了一个没有代码的地址。但是需要注意的是，如果调用的是函数本身就算成功也不返回任何数据的话，那么我们实际上是没有办法区别到底我们成功调用了这个合约还是我们调用了某个 EOA 地址的，所以这种情况下你可能会需要手动检查我们调用的目标地址是否包含代码。


## 什么场景下真的需要这些?

基于三方协议构建的复杂的协议通常会需要在它们的系统中引入一些错误处理（某些 ERC20 代币合约并不遵循标准规范），因此在真实的生产环境中这些错误处理的策略其实非常常见。当然这也取决你有多依赖三方合约以及这些合约的稳定程度，很有可能你根本不需要考虑那么完善，毕竟需要担心外部合约抛出错误的场景算是少数。大部分情况下，啥都不做让错误抛出并向上传递是事实上更简单并且也是完全可接受的处理方式。

## 可以执行的示例

[代码示例](./PooledExecute.sol)展示的是一个用户可以通过 `join()` 方法贡献了 ETH 的合约，一旦募集到了足够的 ETH，那么就可以执行预先设计好的某个外部调用。当然一旦调用失败，每个贡献者都可以通过 `withdraw()` 方法取回自己贡献的 ETH，可以想到如果没有错误处理的话，那么我们就得设计其他方式来开始退钱流程了。

## 参考

这篇文章中只是对 Solidity 中错误以及回滚的相关话题做了简要的描述，如果希望了解更多技术细节可以参看[官方文档](https://docs.soliditylang.org/en/v0.8.16/control-structures.html#error-handling-assert-require-revert-and-exceptions)。

## 一些可能有用的链接
- [EVM Codes - REVERT](https://www.evm.codes/#fd?fork=shanghai)
- [EVM Codes - INVALID](https://www.evm.codes/#fe?fork=shanghai)
- [Solidity by Example  try-catch](https://solidity-by-example.org/try-catch/)
