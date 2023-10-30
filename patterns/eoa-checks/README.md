# 个人钱包地址（非合约地址）检查
- [📜 示例代码](./KingOfTheHill.sol)
- [🐞 测试](../../test/KingOfTheHill.t.sol)

个人钱包地址指的是那些从某私钥经过加密算法产生而来的地址。这些地址不是智能合约，也永远不会成为智能合约地址。说简单点，这些就是最普遍意义上的钱包地址，比如小狐狸钱包地址，冷钱包地址，还有纸钱包地址等等。另一种不同概念的钱包比如Argent和Gnosis Safe是“智能”钱包，他们实际上是定义了像钱包一样的功能的智能合约代码。

开发人员经常需要考虑到正在与之交互的地址是个人地址还是合约地址。了解它们之间的区别与后果是很重要的，尤其是你想要写一个比较完善的对恶意行为有防御性的合约。让我们来快速过一遍这些区别和它们会对你的合约产生什么样的影响。

### 向XXX发起调用
任何试图对一个个人地址发起的函数调用都会成功，但是没有代码会被执行也没有数据被返回。如果这个尝试的函数逻辑本身就不带返回值，那么这种情况就容易跟“试图对真的合约进行函数调用并且真的执行了”有所混淆，因为两种行为都不带返回值。所以要明确被发去调用的地址到底是不是一个智能合约，这是很重要的。

### 被来自于XXX调用
真正的个人地址（由私钥衍生而来的）[目前](https://eips.ethereum.org/EIPS/eip-3074)只能在*每项交易*里发起一条直接的函数调用，然而智能合约发起函数调用时却不受此限。重入攻击，套利交易，预言机操纵，闪电贷攻击等等这些行为若是从一个智能合约的一项交易中发起，就比从个人地址而来的若干项组合交易的完成方式要更加简单可行有利可图。这也是为何那些发生过的重大攻击事件都是先行部署一个用来攻击的智能合约，然后这些攻击行为都在同一条交易中一次性发起。

### ETH转账
在EVM层面，纯粹的ETH转账（例如  `address(receiver).transfer(1 ether)`, `address(receiver).send(1 ether)`, or `address(receiver).call{value: 1 ether}("")`）都会认做一个不含有calldata的空函数调用。正如前面提到过的，任何向EOA发起的调用都会成功并且不执行什么代码。但是如果转账目标实际是一个智能合约，它就会去运行这个合约的字节码，合约方获得了执行代码的权利来做它们设计好要做的事情（在gas够用的前提下）。除了广为人知的用这个机会来做重入攻击，恶意的合约还可以单纯地让交易逆转，这样你的合约的这个函数执行就永远不会成功。

### 代币转账
有的代币标准允许一种转账处理程序，会以接收方名义来发起一个标准的函数调用（例如 `onERC721Received()`）来作为对接收到代币转账的反应（类似于ETH转账可以触发某些代码执行）。所以某些代币转账给智能合约也会具有像ETH转账那样的风险。

### Stuck Assets
Assets (ETH, ERC20s, ERC721s, etc.) held by an EOA are almost always accessible and transferrable by whomever knows the private key. On the other hand, smart contracts are not controlled by a private key. If a contract does not expose functions to directly interact with an asset it holds, they may become permanently stuck in that contract. This is one of the motivations for token standards like `ERC721` and `ERC1155` having "safe" transfer functions that require a contract recipient to respond to an on-transfer hook to signal deliberate support for receiving tokens.

It should be clear by now that interactions with smart contracts are generally considered more risky because their behavior is less predictable and can kick off complex interactions that your protocol may not be designed to handle. But it can sometimes be equally disastrous to interact with an EOA when you expect a contract. For these reasons, some contracts will opt to impose restrictions on the types of accounts they interact with. But how do you identify them?

## `ADDRESS.code.length` Check
It's actually quite simple to check if an address has code in it, and is therefore a contract. Solidity exposes this with the `ADDRESS.code.length` syntax, which returns the (byte) size of the code at that `ADDRESS`. If this value is nonzero, there is a smart contract there.

```solidity
 function _isContractAt(address a) view returns (bool) {
    return a.code.length != 0;
 }
 ```

It's important to understand that this check does not guarantee that the address is an EOA. It only checks if there is code at the address *presently*. It may not be an EOA but a yet-to-be deployed contract address (which are in fact [deterministic](../factory-proofs/)). A contract could even be deployed to that address in the same transaction, right after you've performed this check and lost execution control. It's also possible for a contract to `selfdestruct()` its code away and have it reinstated with `CREATE2`. Therefore, this specific approach is considered a weak EOA check and is usually reserved for non-critical sanity checks or where it only matters that an address is not a contract during a brief call window.

 ## `tx.origin` Check
 This is a reliable and cheap way to guarantee that an address is a *certain* EOA. In solidity, `tx.origin` returns the address that signed the current transaction, which must always be an EOA. Thus, if an address in question is equal to `tx.origin`, you can be sure it's an EOA and will always be one. You will frequently see this check as a modifier on user-facing functions that seek to minimize their attack surface by ensuring that they can only be called by an EOA.
 
 ```solidity
 modifier onlyEOA() {
    require(tx.origin == msg.sender, 'only EOA can call');
    _;
 }
 ```
 
 Keep in mind that this check can only draw a definitive conclusion if the address matches `tx.origin`. Addresses that do not match `tx.origin` can still be EOAs.

 ## Transaction Proofs

 Assuming your contract has access to historical a block hash where an EOA has made a transaction, there's another definitive, albeit obscure and extremely technical, way to check that it is an EOA which doesn't require it to be the current `tx.origin`.

 Every mined Ethereum block is uniquely identified by a "block hash," which is essentially analogous to the hash of the [properties](https://ethereum.org/en/developers/docs/apis/json-rpc/#eth_getblockbyhash) that make up an Ethereum block. The one block property we're most interested here is the `transactionsRoot`. This is a [merkle root](../merkle-proofs/) of all the transactions (actually their hashes) that have been included in that block. Because only EOAs can sign/originate transactions, you can theoretically prove that an arbitrary address is an EOA if you supply proof to your contract that a transaction sent by the address in question is part of a verifiable block hash.
 
 The details of this approach are a bit of a rabbit hole so we won't go much deeper. But in case you do decide to explore it yourself, let me warn you that this approach is only immediately feasible with transactions that were mined in the last [256 blocks](https://docs.soliditylang.org/en/v0.8.17/units-and-global-variables.html#block-and-transaction-properties) on mainnet. Anything older than that, while not impossible, will require extra legwork. 

## Do You Really Need EOA Checks?
These checks are often employed as a security shortcut to mitigate reentrancy, composability attacks (e.g., arbitrage or oracle manipulation), or to just add a speed bump to certain operations (e.g., preventing a contract from minting multiple NFTs in a transaction). Depending on the type of check you use and where you use it, this protection may only be partial. It's important to be aware that EOAs can still perform multiple, direct calls in a single block (but across different transactions), which has been made even easier with the introduction of [flashbot bundles](https://docs.flashbots.net/flashbots-auction/searchers/advanced/understanding-bundles). Still, many developers will continue to opt for this strategy and it's hard to fault them because these checks are fairly cheap, can often reduce the attack surface, and sometimes partial protection is better than none at all, especially in relatively low-stakes applications.

There is a major downside to using EOA checks, and that is in limiting your protocol's composability with other contracts which may try to build on top of yours. Some audiences also have a meaningful share of users that transact from smart wallets, which could be excluded from participating. But not all protocols need to worry about composability, and they may be OK with excluding smart wallet users. It really depends on who you think will be consuming your protocol.

Another point to make is that most attack vectors that EOA checks are used to mitigate can often be better addressed through specific fixes (like reentrancy guards, timelocks, pull patterns, etc), which will not impose restrictions on what kinds of addresses your contract can interact with.

## The Demo
The [demo](./KingOfTheHill.sol) showcases 3 contracts that implement an on-chain king-of-the-hill game. Becoming king costs ETH and lets you set a custom message on the contract. Anyone can become the new king by paying more than the last king did through `takeCrown()`. The amount paid by the new king goes to the old king, and so on.

The naive version, `BrickableKingOfTheHill`, is vulnerable to a denial-of-service attack if the previous king is a contract that reverts when it receives ETH. This causes the call to `takeCrown()` to fail when the payment is sent to the old king (see the [tests](../../test/KingOfTheHill.t.sol#L48) for an example). The result is that the evil contract king remains king forever. The `OnlyEOAKingOfTheHill` version fixes this by simply adding an `onlyEOA` modifier to `takeCrown()`. This ensures that the every king is, and always will be, an EOA, which cannot revert on ETH transfers. The `ClaimableKingOfTheHill` version also fixes this by only sending ETH to the last king if they have no code at their address, otherwise it will set aside the ETH for the last king to `claim()` it in a separate call.
