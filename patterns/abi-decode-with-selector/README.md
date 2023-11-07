# 带选择器的 ABI-encoding 数据的解码
- [📜 示例代码](./ApproveRestrictedWallet.sol)
- [🐞 测试](../../test/ApproveRestrictedWallet.t.sol)

[ABI-encoding](https://docs.soliditylang.org/en/v0.8.19/abi-spec.html#argument-encoding) 应用在了几乎所有的 EVM 数据传递场景，它可以将任意数据类型编码为原始的字节数组（bytes）以方便进行数据传递，例如函数调用，错误抛出以及非索引事件数据中其实底层都有使用到 ABI-encoding。内置的函数 `abi.encode()` 就是其对应在 Solidity 中的实现，该函数可以帮助你编码任意数据。

```solidity
// Encode a (uint256,address) tuple
bytes memory encoded = abi.encode(
    uint256(123),
    address(0xF0e20f3Be40923b0C720e61A75Fb6940A3929019)
);
// encoded == hex"000000000000000000000000000000000000000000000000000000000000007b000000000000000000000000f0e20f3be40923b0c720e61a75fb6940a3929019"
```

与 `abi.encode()` 相对应的逆操作解码函数是 `abi.decode()`，它接受 ABI-encoding 编码过的字节数组以及该数组实际的数据类型，进行解码后返回其原始的数据值。

```solidity
// Decodes to x = 123, a = 0xF0e20f3Be40923b0C720e61A75Fb6940A3929019
(uint256 x, address a) = abi.decode(encoded, (uint256, address));
```

但当我们需要发起/处理底层函数调用或是抛出错误的时候，实际上第一件事应该是确认该编码对应的函数签名或是错误类型，这样我们才能根据实际的数据类型对数据进行编码/解码。这就是为什么，无论是函数调用还是抛出错误的 ABI-encoding 数据都需要在其开头附加上四字节的“选择器”来表明函数或是错误的类型。Solidity 中的内置函数 `abi.encodeWithSelector()` 以及 `abi.encodeCall()` 就是用来做这个事情的，它在 `abi.encode` 的基础上在返回的字节数组的开头添加了四字节的选择器。

当然，Solidity 中并没有内置 `abi.encodeWithSelector()` 的逆操作函数，这并不代表该操作是不可实现的, 接下来这个场景就需要通过解析带选择器的 ABI-encoding 数据来完成。

## 案例: 限制 approve() 调用

假设我们正在开发一个智能合约钱包，钱包合约中有一个 `exec()` 函数，它接受任意合约函数调用对应的 ABI-encoding 后的字节数组作为参数，然后由钱包合约带着参数对目标合约发起调用，效果上就好像是这个调用就是直接从智能合约钱包发起的一样。

```solidity
function exec(address payable callTarget, bytes calldata fnCallData, uint256 callValue)
    external
    onlyOwner
{
    (bool s,) = callTarget.call{value: callValue}(fnCallData);
    require(s, 'exec failed');
}
```

也许我们会希望只允许这个钱包授权它的转账权限给我们提前确认过的合约，否则攻击者只需要诱导用户将钱包的转账权限授权给攻击者部署的恶意合约即可转走用户的资产。为了实现这一点，我们需要解析 `fnCallData` 这个参数，如果检测到实际发起的调用是某个 ERC20 合约的 `approve()` 方法，那么这时候 approve 授权的对象必须在我们提前确认过的合约列表中。上述过程的具体步骤如下：

1. 解析开头的四字节选择器。
2. 如果选择器正好等于 `ERC20.approve`，那么继续解析其参数。
    1. 确认解码出来的参数中的 `spender` 字段是否在我们提前确认过的列表中。


因为 `fnCallData` 参数是位于 `calldata` 上而不是 `memory` 上，所以整体的代码非常简洁，具体原因会在下一小节解释：

```solidity
// Compare the first 4 bytes (selector) of fnCallData.
if (bytes4(fnCallData) == ERC20.approve.selector) {
    // ABI-decode the remaining bytes of fnCallData as IERC20.approve() parameters
    // using a calldata array slice to remove the leading 4 bytes.
    (address spender, uint256 allowance) = abi.decode(fnCallData[4:], (address, uint256));
    require(isAllowedSpender[spender], 'not an allowed spender');
}
```

### `memory` 的缺陷
上面的简洁实现主要得益于 Solidity 中的 `calldata` 对于[数组切片（也就是 `[4:] 的语法`）的原生支持，可惜该切片操作*仅仅*适用于 `calldata` 数组。换句话说如果 `fnCallData` 是在 `memory` 上而不是 `calldata` 上那么我们就没办法使用切片操作。

因此这时候你需要手动拆分出 ABI-decoding 需要的部分（🤮），要么把数组剩下的部分（除开四字节选择器以外的部分）拷贝下来传给 `abi.decode()` （💸），要么暂时原地修改数组去掉选择器的部分然后将其传给  `abi.decode()` （🤗）。下面我们来展示第二种方式，因为它会更有意思：

```solidity
// Note that now fnCallData is in memory.
function exec(address payable callTarget, bytes memory fnCallData, uint256 callValue)
    external
    onlyOwner
{
    // Compare the first 4 bytes (selector) of fnCallData.
    if (bytes4(fnCallData) == ERC20.approve.selector) {
        // Since fnCallData is located in memory now, we cannot use calldata slices.
        // Modify the array data in-place to shift the start 4 bytes.
        bytes32 oldBits;
        assembly {
            let len := mload(fnCallData)
            fnCallData := add(fnCallData, 4)
            oldBits := mload(fnCallData)
            mstore(fnCallData, sub(len, 4))
        }
        // ABI-decode fnCallData as ERC20.approve() parameters. 
        (address spender, uint256 allowance) = abi.decode(fnCallData, (address, uint256));
        // Undo the array modification.
        assembly {
            mstore(fnCallData, oldBits)
            fnCallData := sub(fnCallData, 4)
        }
        require(isAllowedSpender[spender], 'not an allowed spender');
    }
    // rest of function ...
}
```

上述技巧的原理是依赖 `memory` 中的 `bytes` 数组变量的前 32 个字节存储的是数组的长度，其后才是数组的具体内容，上述过程我们相当于对数组指针进行了 32 字节的偏移，这时候原本存储长度的位置被忽略，而存储选择器的四个字节将负责存储数组的长度，然后我们再将其更新为数组长度减 4，正好对应了剩余的字节长度。

![memory layout for bytes array](./array.drawio.png)

## 示例代码

[代码文件](./ApproveRestrictedWallet.sol)包含了上述案例中智能合约钱包的代码，其中包含了 2 组不同的实现，他们继承了相同的基类合约，唯一的区别是两者的 `fnCallDara` 分别是声明在了 `calldata` 和 `memory` 上。


## 一些可能有用的链接
- [abi-encode 相关的函数用法示例](https://solidity-by-example.org/abi-encode/)
- [calldata 关键字可以被用在任意可见性的函数中吗](https://ethereum.stackexchange.com/questions/123169/can-calldata-be-used-in-every-function-visibility)

