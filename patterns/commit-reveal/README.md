# 先提交-再揭晓

- [📜 示例代码](./SealedAuctionMint.sol)
- [🐞 测试](../../test/SealedAuctionMint.t.sol)

每一个在以太坊链上被确认并记录的交易都是一条对任何人公开可见的，永久性的记录。即使是此刻暂未上链的，存在于公开的以太链交易池中等待被验证者打包上链的那些交易也同样是对任何人公开可见的。这种区块链固有的公开透明的特性令其具有便利性和可靠性；然而，此种公开如同一把双刃剑，因为允许任何用户去知晓其他用户即将去做或者已经做了什么交易，都有可能会鼓励一些有意的反向行为（比如说抢跑或跟随交易，恶意阻击拍卖等等）在链上发生，进而可以影响你的代码协议在使用中的公平性。

如果我们能够找到一种方式来“秘密地”在以太链上做事，则可以大大缓解上述的那些问题。一种“先提交-再揭晓”的模式不失为一个简单的解决方案，有些协议已经开始应用了这个模式：它允许用户先行提交一个“密封起来的”链上交易，这个交易会在以后的某一时刻被执行。此类的协议需要被设计成具有分开的“提交”和“揭晓”不同阶段，令用户可以去相应地提交两个匹配的交易：

### “先提交”交易

在这个“提交”阶段，用户率先向协议发起一个“提交”交易，此交易会绑定给随后在“揭晓”阶段发起的另一条交易来完成某特定的行为。这个“提交”交易通常为一个哈希值，是将那个后来需要被完成的行为交易的具体细节加上一个用户自定义的“料”合在一起被`hash`的结果（例如，`commit = keccak256(ACTION, SALT)`）。因为哈希函数可被认为单方向单一映射并不可逆向计算，那么如果不知道这个“料”的值，则基本上不可能获知是哪一个具体行为交易被用来产生此被提交的哈希值的。


### “再揭晓”交易

在揭晓的阶段，用户必须揭晓当初被用来生成那个哈希的具体的行为细节。用户需向协议发起第二个交易，此交易即“揭晓”交易，它会包含与之前那个哈希的产生所相应的行为细节信息以及那个“料”。然后协议会基于此行为信息及“料”的值来重新计算哈希，仅当这个计算值与之前被提交的那个哈希值一致时，才会去将这个特定的行为赋予执行。因为任何被提交的具体行为必须伴有一个之前的“提交”交易（哈希）方可被执行，那么诸如抢跑交易之类的行为在可行性上就被大打折扣。

## 案例分析: 保密拍卖
我们用一个NFT合约（非严格ERC721）来示范这种模式怎样起作用：这个NFT合约以“1天”做分隔单位来实行保密拍卖机制，所拍卖的是一份可以铸造一个新币的权利。我们称呼这个协议为"Nowns" 😉。

每24小时，一轮新的拍卖开启，任何人都可以发起一个保密的`bid()` (即“提交”阶段)，在交易中发送的ETH要比真实的拍卖叫价高一些来混淆真实意图。所提交的 `commitHash` 应该用函数 `keccak(bidAmount, salt)`，此间的 `bidAmount` 和 `salt` 是保密的只有此发起人知晓其值，并且 `bidAmount <<< msg.value`。

```solidity
function bid(uint256 auctionId, bytes32 commitHash) external payable {
    require(auctionId == getCurrentAuctionId(), 'auction not accepting bids');
    require(commitHash != 0, 'invalid commit hash');
    require(bidsByAuction[auctionId][msg.sender].commitHash == 0, 'already bid');
    require(msg.value != 0, 'invalid bid');
    bidsByAuction[auctionId][msg.sender] = SealedBid({
        ethAttached: msg.value,
        commitHash: commitHash
    });
}
```

24小时后，“提交”阶段结束，拍卖进入“揭晓”阶段，任何之前有过提交交易的用户现在可以有24小时时限来发起 `reveal()` 揭晓交易。

```solidity
function reveal(uint256 auctionId, uint256 bidAmount, bytes32 salt) external {
    require(auctionId < getCurrentAuctionId(), 'bidding still ongoing');
    require(!isAuctionOver(auctionId), 'auction over');
    SealedBid memory bid_ = bidsByAuction[auctionId][msg.sender];
    // 确保之前被提交的哈希与现在通过竞拍和“料”的值计算所产生的哈希一致
    require(bid_.commitHash == keccak256(abi.encode(bidAmount, salt)), 'invalid reveal');
    uint256 cappedBidAmount = bidAmount > bid_.ethAttached
        ? bid_.ethAttached : bidAmount;
    // 如果此用户叫价高于目前的最高叫价，则此用户为新的胜出者。
    uint256 winningBidAmount = winningBidAmountByAuction[auctionId];
    if (cappedBidAmount > winningBidAmount) {
        // Caller is the new winning bidder.
        winningBidderByAuction[auctionId] = msg.sender;
        winningBidAmountByAuction[auctionId] = cappedBidAmount;
    }
}
```

在这48小时时段之后，竞拍结束。胜出者被赋予可执行 `mint()` 函数的权限来给自己铸一个新币，同时退还当时发送的ETH多余的差价。

```solidity
function mint(uint256 auctionId) external {
    require(isAuctionOver(auctionId), 'auction not over');
    address winningBidder = winningBidderByAuction[auctionId];
    require(winningBidder == msg.sender, 'not the winner');
    SealedBid storage bid_ = bidsByAuction[auctionId][msg.sender];
    uint256 ethAttached = bid_.ethAttached;
    require(ethAttached != 0, 'already minted');
    // 将 `ethAttached`值修改为0来避免重复铸币
    bid_.ethAttached = 0;
    _mintTo(msg.sender);
    // 退还多余ETH资金
    uint256 refund = ethAttached - winningBidAmountByAuction[auctionId];
    payable(msg.sender).transfer(refund);
}
```

在“提交”阶段的24小时以外的任何时刻，任何参与竞拍着都可以试着发起 `reclaim()` 函数来撤销掉叫价并取回资金，但只有除当时时刻的的胜出者以外的其他“失败者”能够成功执行这一操作。

```solidity
function reclaim(uint256 auctionId) external {
    require(auctionId < getCurrentAuctionId(), 'bidding still ongoing');
    address winningBidder = winningBidderByAuction[auctionId];
    require(winningBidder != msg.sender, 'winner cannot reclaim');
    SealedBid storage bid_ = bidsByAuction[auctionId][msg.sender];
    uint256 refund = bid_.ethAttached;
    require(refund != 0, 'already reclaimed');
    // 将 `ethAttached`值修改为0来避免重复撤回。
    bid_.ethAttached = 0;
    payable(msg.sender).transfer(refund);
}
```

面对类似这样的盲拍合约，用户基本难以去做拍卖狙击（比目前最高叫价加一块钱儿的）或者抢跑任何叫价交易，因为真实的叫价数值是保密的，直至被揭晓时刻，而被揭晓的时候恶意竞争者已经无法再成功发起叫价交易了。完整的合约代码示例 [在这里](./SealedAuctionMint.sol)，这里是 [测试代码](../../test/SealedAuctionMint.t.sol).

## 现实中的应用
[Ethereum Name Service](https://ens.domains/) (ENS) 大概是采用这种方式的协议里最出名的那个了。[最初版本的注册合约](https://etherscan.io/address/0x6090a6e47849629b7245dfa1ca21d94cd15878ef#code) 即针对ENS域名采取盲拍方式，并且正如我们的例子这样应用了“先提交-再揭晓”的执行模式。[新版合约](https://docs.ens.domains/contract-api-reference/.eth-permanent-registrar/controller) 则不再采用竞拍的机制，但依然使用着“先提交-再揭晓”的执行模式（保护正在被购买的域名）来防止抢跑购买行为。

