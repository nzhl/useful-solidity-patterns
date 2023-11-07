# EIP712 签名

- 示例代码:
    - [📜 合约](./MintVouchers.sol)
    - [🌐 前端](https://codesandbox.io/s/compassionate-dust-jgeydc?file=/src/App.vue)
    - [🐞 测试](../../test/MintVouchers.t.sol)

[EIP712 标准](https://eips.ethereum.org/EIPS/eip-712)定义了一种令用户可以用其私钥对任何人类可理解的 JSON 格式的信息进行电子签名。在小狐狸钱包里，会弹出一个窗口并且列出这个信息的各项内容供用户在确认签名之前阅读：

![metamask EIP712 popup](./metamask-721.png)

这个签名行为被视为用户在链下确认过被签署的信息内容。数字签名可以在链下被分享，并且以后被（其他角色）用在链上来执行某交易，此交易被认为已被签过名的用户钱包授权。

## 案例分析: 投票

我们通过一个简易的治理协议来展示一下分布式应用及其后端合约是如何工作的。这个协议允许人们针对某项提议通过其 `proposalId` 来发送链上投票。为了简化，这个协议允许任何人针对任何提议投票，并且只对赞成票来计数。

首先我们来看一个完全在链上执行的合约版本，然后不断改进它。这一版本带有一个 `voteFor()` 函数，任何投票者都需要直接调用这个函数来进行投票操作。

```solidity
mapping (uint256 => mapping (address => bool)) public hasVotedForProposalId;
mapping (uint256 => uint256) public yesVotesForProposalId;

...

function voteFor(uint256 proposalId) external {
    // 确保此人之前没有对这项提议投过票。
    require(!hasVotedForProposalId[proposalId][msg.sender], 'already voted');
    hasVotedForProposalId[proposalId][msg.sender] = true;
    // 增加这项提议的赞成数。
    ++yesVotesForProposalId[proposalId];
}
```

利用 EIP712 标准，就可以让用户无需花费 gas 来投票。投票者对一条链下的投票消息进行签名，这个消息表达了其想要对某项提议投赞成票的意愿。随后另外一角色可以通过调用 `voteForBySignatures()` 函数来将所有的投票签名进行归集并且一次性地计数到链上。这个函数接受的输入值为列表类型的投票者们地址，其所投的提议号，还有电子签名的各项组成部分。函数会针对每一个投票输入执行如下步骤：

1. 确保此人此前并未对此提议投过票。
2. 计算对这条投票消息的 EIP712 型哈希值。
3. 利用内置的 `ecrecover()` 预编译函数，通过提供的电子签名各项以及此投票消息的哈希值来计算得到签名者的地址，并校验此地址与提供的投票者地址是否一致来确认投票有效性。若投票者在应用前端所看到的投票信息与真实信息有任何不同（导致哈希值变化）或者传输过程中数字签名的任意项被篡改，那么这个计算都不会得到与已知一致的地址，验证失败，投票无效。
4. 若验证通过，则链上增加相应提议的赞成数。

```solidity
function voteForBySignatures(
    address[] calldata voters,
    uint256[] calldata proposalIds,
    uint8[] calldata vs,
    bytes32[] calldata rs,
    bytes32[] calldata ss
)
    external
{
    for (uint256 i = 0; i < voters.length; ++i) {
        uint256 proposalId = proposalIds[i];
        address voter = voters[i];
        require(!hasVotedForProposalId[proposalId][voter], 'already voted');
        hasVotedForProposalId[proposalId][voter] = true;
        require(voter == ecrecover(_getVoteHash(proposalId), vs[i], rs[i], ss[i]), 'bad signature');
        ++yesVotesForProposalId[proposalId];
    }
}
```

### EIP72 型哈希

像小狐狸这样的钱包应用会接受消息的各项内容，计算其哈希并且用当前地址的私钥对这个哈希值进行签名。这里的哈希计算有其特定的方式，是根据 [EIP712 的要求](https://eips.ethereum.org/EIPS/eip-712#specification)来确保来自于不同协议的消息一定不会产生哈希碰撞。在这里我们用之前调用过的 `_getVoteHash()` 函数来在链上重新计算消息的哈希值。

```solidity
function _getVoteHash(uint256 proposalId) private view returns (bytes32 hash) {
    // 计算此协议所在域的哈希，应具有唯一性来指向此已部署的协议.
    bytes32 domainHash = keccak256(abi.encode(
        keccak256('EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)'),
        // 协议的名字
        keccak256('ExampleVotingContract'),
        // 协议版本
        keccak256('1.0.0'),
        // 协议所在的链号
        block.chainid,
        // 此协议被部署的标准地址
        address(this)
    ));
    // 此消息类型相关的哈希值
    bytes32 structHash = keccak256(abi.encode(
        keccak256('Vote(uint256 proposalId)'),
        proposalId
    ));
    return keccak256(abi.encodePacked('\x19\x01', domainHash, structHash));
}
```

 `domainHash` 和消息类型的哈希值是不会更改的，所以你可以（并应当）用常量来存放它们。

### 从用户处获取投票签名

在前端，我们的应用会请求已链接的钱包应用生成一个对投票消息的电子签名。可以使用 `ethers` 库里的[`Signer._signTypedData()`](https://docs.ethers.io/v5/single-page/#/v5/api/signer/-%23-Signer-signTypedData) 方式对一个钱包建立的 “Signer” 实例发起这样的请求。需要传入的变量是：

1. 域对象，其各项内容要与 `_getVoteHash()` 中所使用的相一致。
2. 消息里包含的各项 EIP712 定义类型的键值表示，最后一项应是我们的根消息类型，要与 `_getVoteHash()` 中所使用的相一致。
3. 一个对象来提供消息中的各项所需值。在我们的例子中的消息仅需要一项，`proposalId`。

```js
// 小狐狸钱包已连接。
const {v, r, s} = ethers.utils.splitSignature(
    await provider.getSigner()._signTypedData(
        // 域
        {
            name: 'ExampleVotingContract',
            version: '1.0.0',
            chainId: 1, // 以太坊L1主网
            verifyingContract: DEPLOYED_VOTING_CONTRACT_ADDRESS,
        },
        // 型
        [ { Vote: [ { name: 'proposalId', type: 'uint256' } ] } ],
        // 消息
        { proposalId: YOUR_PROPOSAL_ID },
    ),
);
```

这样则会弹出一个窗口请求用户对消息 `proposalId: YOUR_PROPOSAL_ID` 进行签名，并且以串联 16 进制字符串的形式返回电子签名的各项内容。我们使用 `splitSignature()` 来解构这个字符串为 `v`， `r`， 和 `s` 三项内容供 `voteForBySignatures()` 使用。

## 优缺点

#### 优点: 不花 gas 的前端交互
想象一下你仅需对一个链下消息进行签名来授权你对某种链上行为和其触发条件的同意。这个授权平时被存在链下，直至链上行为的触发条件已达成，这个授权会被他人用来传递给协议来执行你之前已同意的行为并无需你本人再次来操作什么。因为执行交易是别人发起的，所以你不用花钱儿付 gas。

#### 优点: 总体来说用户更省 gas
这个方式很切合[链下数据存储](./off-chain-storage)的要求，因为消息中各项信息都汇集到一个哈希值里。加之，这个被签署过的消息带来的额外好处就是仅在需要的时候才会在链上写入数据。消息因为已经被签过字所以是可信的。

#### 优点: 归集所有交易统一处理
所有的签过字的消息都可以被归集起来，在一个交易内统一处理。

#### 优点: 利用链下规模化
所有消息的统筹和归集都可以在链下进行，并可以利用 Web2 的一些现有优势。只有最后的交易实现那步需要上链。

#### 缺点: 被审核处理的风险
因为此签名非链上行为，它通常会在链下以传统的 Web2 渠道来传播和存放在某一中心化个体处（例如某个后端数据服务器）。虽然没有人能够篡改这个经过签名加密的消息内容，但是可以选择将其隐藏不示于人，令其不起作用。这是一种实打实的被中心化审核处理的风险。

#### 缺点: 可取消
签名行为是不可逆的。唯一能够撤销某一行为的方法是在链上做一些反制行为。一些典型的协议，如带有取消功能，一种简单的实现就是将某消息标记为“已执行/已取消”之类来防止它以后被执行。另一种处理方式是在消息内加上一个过期时间戳，这个时间戳的值会在此消息的链上执行时被验证有效性。去做一个长期有效的消息，并在想要取消的时候发起一条取消交易的方式成本较高；而去做一个短期有效的消息，如果消息过期则发起一个新消息来替代已过期的旧的，这种方式显然成本要低。

#### 缺点: 分给额度/替代保管
若交易涉及到此协议之外的资产，用户通常需要先将资产存在协议处，或划分给协议可处理特定数量自己的此种资产的额度。因为签字的用户不会是日后来发起执行交易的人，所以协议需要上述方式存在作为可执行的前提。

## 显著用途

这种模式已经在 DeFi 里很常见了，同时其他一些板块和领域也在逐渐采取它。


- [0x](https://docs.0x.org/), [Opensea](https://support.opensea.io/hc/en-us/articles/4449355421075-What-does-a-typed-signature-request-look-like-)（seaport 和 wyvern），[CoWSwap](https://docs.cow.fi/smart-contracts/settlement-contract/signature-schemes)，等等......
    - 这些协议均采用此种 EIP712 方式获取限价下单的授权，这个单子随后可以被对方卖家来发起执行。
- [Uniswap](https://github.com/Uniswap/governance/blob/master/contracts/GovernorAlpha.sol#L248) 和 [Compound](https://docs.compound.finance/v2/governance/#cast-vote-by-signature) Governance
    - 会员可以投票或委托投票权给他人，均用 EIP712 链下签名方式。
- [Opensea Lazy-minting](https://opensea.io/blog/announcements/introducing-the-collection-manager/)
    - 某 NFT 系列的所有者可以签 EIP712 来授权其他人，当买卖交易达成时，允许其铸造次被交易的通证。
- [ERC20 Permit Extension](https://eips.ethereum.org/EIPS/eip-2612)
    - 这是一个很受欢迎的授权拓展，利用 EIP712 链下签名，授权他人使用己方资产额度的时候替代了 Approve 这一步需要花费 gas 的链上操作。

## 例子

[这个例子](./MintVouchers.sol)是一个限定了铸币权限的 ERC721 协议。协议的部署者可以通过签 EIP712 消息的方式来发“券”，这个券写明了一个通证号还有想要铸造此通证需要多少钱。用户可以选择兑换此券来发起一条交易施行铸币的权利，但需要他们交付在券里规定好的 ETH 数量。这个交易同时会将这些附上的 ETH 发送给部署者。如果他对于某一已经发出的券上所写的交易金额后悔了想做修改，那么他可以调用 `cancel()` 函数，这个券就不能被他人兑换了。

此例含有两个部分: [合约](./MintVouchers.sol)和[前端](https://codesandbox.io/s/compassionate-dust-jgeydc?file=/src/App.vue)运行在 CodeSandbox 上。这个前端已经整合了被预搭建好的合约主体。当前端与你的小狐狸钱包链接后，它会允许你部署合约，签字发布新的券，兑换签过字的券。这些功能都在同一页上。

## 资料
- [EIP712 相关信息](https://eips.ethereum.org/EIPS/eip-712)
- [ethers.js Signer 文档](https://docs.ethers.io/v5/single-page/#/v5/api/signer/-%23-Signer-signTypedData)
