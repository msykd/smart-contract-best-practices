以下は、スマートコントラクトを作成する際に注意し、防御する必要がある既知の攻撃の一覧です。

## Reentrancy

外部コントラクトを呼び出す際の主要な危険の1つは、コントロールフローを引き継ぎ、呼び出し関数が期待していなかったデータを変更できることです。
このクラスのバグはいろいろな形を取ることができ、DAOの崩壊につながった主要なバグの両方がこの種のバグでした。

### Reentrancy on a Single Function¶

このバグの最初のバージョンには、関数の最初の呼び出しが終了する前に、繰り返し呼び出される可能性のある関数が含まれていました。
これにより、関数のさまざまな呼び出しが破壊的なやり方で相互作用する可能性があります。

```sol
// INSECURE
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call withdrawBalance again
    userBalances[msg.sender] = 0;
}
```

関数の最後までユーザーの残高が0に設定されていないため、2回目以降の呼び出しは引き続き成功し、何度も何度も残高を回収します。
非常によく似たバグが、The DAO攻撃の脆弱性の1つでした。

この問題を避ける最も良い方法は、[`call.value()()`の代わりに`send()` を使うことです](https://github.com/ConsenSys/smart-contract-best-practices#send-vs-call-value)。
これにより外部コードの実行を防ぎます。

ただし、外部呼び出しを削除できない場合、この攻撃を防ぐ最も簡単な方法は、必要な内部作業をすべて完了するまで外部関数を呼び出さないようにすることです。

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // The user's balance is already 0, so future invocations won't withdraw anything
}
```

`withdrawBalance()`と呼ばれる別の関数があった場合、それは潜在的に同じ攻撃の対象となるため、
信頼できないコントラクトを呼び出す関数を信頼できないものとして扱わなければなりません。 潜在的な解決策の詳細については、以下を参照してください。

### Cross-function Reentrancy

攻撃者は同じ状態を共有する2つの異なる機能を使用して同様の攻撃を行うこともできます。

```sol
// INSECURE
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // At this point, the caller's code is executed, and can call transfer()
    userBalances[msg.sender] = 0;
}
```

この場合、攻撃者は `withdrawBalance`で外部呼び出しでコードが実行されるときに` transfer() `を呼び出します。
残高はまだ0に設定されていないので、すでに引き出しを受けていてもトークンを転送することができます。この脆弱性は、The DAOへの攻撃にも使用されていました。

同じ解決策が、同じ警告で動作します。
この例では、両方の関数が同じコントラクトの一部であったことにも注意してください。
ただし、同じコントラクトが複数のコントラクトを共有している場合、複数のコントラクトで同じバグが発生する可能性があります。

### Pitfalls in Reentrancy Solutions

再入可能性は複数の関数、さらには複数のコントラクトにまたがって発生する可能性があるため、単一の関数で再入可能性を防ぐことを目的とした解決策では不十分です。

そうではなく、最初にすべての内部作業（つまり、状態の変更）を終了してから、外部関数を呼び出すことを推奨します。
この規則を慎重に守れば、再入可能性による脆弱性を避けることができます。
しかし、外部関数をあまりにも早く呼び出さないようにするだけでなく、外部関数を呼び出す関数を呼び出さないようにする必要があります。
たとえば、以下は安全ではありません。

```sol
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdrawReward(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

`getFirstWithdrawalBonus()`が外部のコントラクトを直接呼び出すことはありませんが、`withdrawReward()`を呼び出すだけで、再入可能性の影響を受けやすくなります。
したがって、あなたは`withdrawReward()`をあたかもそれが信頼できないものとして扱う必要があります。

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdrawReward(address recipient) public {
    uint amountToWithdraw = rewardsForA[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdrawReward(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

再入が不可能であることに加えて、[信頼できない関数がマークされてます。](https://github.com/ConsenSys/smart-contract-best-practices#mark-untrusted-contracts)
この同じパターンは、すべてのレベルで繰り返されます。
`untrustedGetFirstWithdrawalBonus()`は外部コントラクトを呼び出す `untrustedWithdrawReward()`を呼び出すので、
`untrustedGetFirstWithdrawalBonus()`も安全でないものとして扱わなければなりません。

しばしば提案される別の解決策は、[mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)です。
これにより、ロックをしたオーナーだけが変更できるように、状態を「ロック」することができます。 簡単な例は次のようになります。

```sol
// Note: This is a rudimentary example, and mutexes are particularly useful where there is substantial logic and/or shared state
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    if (msg.sender.call(amount)()) { // Normally insecure, but the mutex saves it
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

ユーザーが最初の呼び出しが終了する前に `withdraw()`をもう一度呼び出そうとすると、ロックはそれが効果を発揮するのを防ぎます。
これは効果的なパターンかもしれませんが、協力が必要な複数のコントラクトがある場合は面倒です。
以下は安全ではありません。

```sol
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == address(0));
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = address(0);
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

攻撃者は `getLock()`を呼び出し、 `releaseLock()`を決して呼び出すことはできません。
彼らがこれをすると、コントラクトは永遠にロックされ、それ以上の変更はできません。
再入可能性を防ぐためにmutexを使用する場合は、ロックを要求して解放する方法がないことを慎重に確認する必要があります。
(デッドロックやライブロックのようなmutexを使ってプログラミングするときには、他にも潜在的な危険性がありますので、mutexに書かれている大量の文献を参考にしてください。


## Front Running (別名 Transaction-Ordering Dependence)

上記は、攻撃者が1つのトランザクション内で悪意のあるコードを実行することを含む再入可能性の例です。
以下は、ブロックチェーンに固有の異なる種類の攻撃、つまりトランザクション自体の順序（ブロック内）は簡単に操作されるという事実があります。

トランザクションはしばらくの間mempool内にあるので、ブロックに含まれる前に、どのようなアクションが発生するのかを知ることができます。
これは、いくつかのトークンを購入する取引が見られる分散型マーケットや、他の取引が含まれる前に実行される成行注文のようなものにとっては面倒なことがあります。
特定のコントラクト自体に起因するため、これに対する保護は困難です。
たとえば、マーケットでは、一括オークションを実装することをお勧めします（これにより、高頻度の取引の懸念からも保護されます）。
事前コミットスキームを使用する別の方法もあります。後で詳細を記します。

## Timestamp Dependence

ブロックのタイムスタンプはマイナーによって操作される可能性があることに注意してください。そしてタイムスタンプのすべての直接的および間接的な使用が考慮されるべきです。
タイムスタンプの依存関係に関する設計上の考慮事項については、[推奨する実装方法](./recommendations/#_20)セクションを参照してください。

## Integer Overflow and Underflow

簡単なトークン転送を想定します。

```sol
mapping (address => uint256) public balanceOf;

// INSECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance */
    require(balanceOf[msg.sender] >= _value);
    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// SECURE
function transfer(address _to, uint256 _value) {
    /* Check if sender has balance and for overflows */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* Add and subtract new balances */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

バランスが最大uint値（2 ^ 256）に達すると、ゼロに戻ります。これにより、その状態がチェックされます。
これは、実装に応じて適切かどうかは関係ありません。
uint値にこのような大きな値に近づく機会があるかどうかについて考えてみましょう。
uint変数がどのように状態を変え、そのような変更を行う権限を持っているかについて考えます。
uint値を更新する関数を呼び出すことができるユーザーがいる場合、攻撃を受けやすくなります。
管理者だけが変数の状態を変更するためのアクセス権を持っている場合、あなたは安全かもしれません。
ユーザーが一度に1だけインクリメントできる場合は、この制限に達する実現可能な方法がないため、おそらく安全です。

アンダーフローについても同様です。 uintが0より小さくなると、アンダーフローが発生し、最大値に設定されます。
uint8、uint16、uint24 ...などの小さなデータ型には注意してください。さらに簡単に最大値に達することができます。

[オーバーフローとアンダーフローが発生するケースは約20件あります。](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

### Underflow in Depth: Storage Manipulation

 [Doug Hoyteによる2017年のunderhanded solidity contest（悪意のあるコードを見つけるためのプログラミングコンテスト）への応募](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte)が、[名誉ある賞を受賞](http://u.solidity.cc/)しました。
 このエントリは、CのようなアンダーフローがSolidityストレージにどのように影響するかについての懸念を提起するので、興味深いものです。以下に単純化されたバージョンを示します：

```sol
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }
    
    uint[] public bonusCodes;
    
    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }
    
    function popBonusCode()  {
        require(bonusCodes.length >=0);  // this is a tautology
        bonusCodes.length--; // an underflow can be caused here
    }
    
    function modifyBonusCode(uint index, uint update)  { 
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // write to any index less than bonusCodes.length
    }
    
}
```

一般的に言って、変数`manipulateMe`の位置は、`keccak256`を通さない限り影響を受けることはできません。影響を及ぼすことは実行不可能です。
ただし、動的配列は順次格納されるため、悪意のある行為者が`manipulateMe`を変更したい場合は、次のようにします：
 
 * `popBonusCode`をアンダーフローするために呼び出します（注：`array.pop())`メソッドはSolidity 0.5.0で[追加されました](https://github.com/ethereum/solidity/blob/v0.5.0/Changelog.md)）
 * `manipulateMe`の保管場所を算出します。
 * `modifyBonusCode`を使用して`manipulateMe`の値を変更および更新する

 実際には、この配列はすぐに厄介であると指摘されます。しかし、より複雑なスマートコントラクトアーキテクチャの下に埋め込まれていると、定数への悪意のある変更を勝手に許可する可能性があります。

動的配列の使用を検討する際には、コンテナー・データ構造が良い方法です。 Solidity CRUD[part 1](https://medium.com/@robhitchens/solidity-crud-part-1-824ffa69509a)と[part 2](https://medium.com/@robhitchens/solidity-crud-part-2-ed8d8b4f74ec)の記事は良い情報源です。

<a name="dos-with-unexpected-revert"></a>

## DoS with (Unexpected) revert

簡単なオークションのコントラクトを想定します。

```sol
// INSECURE
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // Refund the old leader, if it fails then revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

古いリーダーを払い戻そうとすると、払い戻しが失敗した場合に元のリーダーに戻ります。
これは、悪意のある入札者が、そのアドレスへの払い戻しが常に失敗することを確実にしながら、リーダーになれることを意味します。
このようにして、他の誰かが `bid()`関数を呼び出すのを防ぐことができ、リーダーを永遠にとどめることができます。
前に説明したように、[プル型の支払い](./recommendations#_8)を代わりに設定することをお勧めします。

もう1つの例は、コントラクトによってユーザー(例えば、クラウドファンディングコントラクトの支援者)に支払うために配列を反復する場合です。
それぞれの支払いが成功することを確認することが一般的です。
そうでない場合は、元に戻す必要があります。問題は、1つの呼び出しが失敗した場合、支払いシステム全体を元に戻すことです。
つまり、ループは完了しません。 1つのアドレスで強制的にエラーが発生するため、誰も支払いを受けません。

```sol
address[] private refundAddresses;
mapping (address => uint) public refunds;

// bad
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // arbitrary length iteration based on how many addresses participated
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // doubly bad, now a single failure on send will hold up all funds
    }
}
```

ここでも推奨の解決策は [プッシュ型よりもプル型が望ましい](./recommendations#_8)ことになります。

## DoS with Block Gas Limit

各ブロックには、使用できるガスの量、つまり実行できる量の計算に上限があります。
これがブロックガスリミットです。消費されたガスがこの制限を超えると、トランザクションは失敗します。これは、DoSの可能性へとつながります。

### 無制限の操作によるコントラクトのガスリミットDoS

あなたは前の例に別の問題があることに気づいたかもしれません。一度に全員に払い戻すことで、ブロックガス制限にぶつかる危険性があります。

これは、意図的な攻撃がなくても問題につながる可能性があります。
しかし、攻撃者が必要なガスの量を操作できるならば、特に悪いことです。
前の例の場合、攻撃者はアドレスの束を追加することができ、それぞれは非常に小さな払い戻しを必要とします。
したがって、各攻撃者のアドレスを払い戻すためのガスコストは、ガス制限を超えてしまい、払い戻し取引がまったく起こらないようになる可能性があります。

これが[プッシュ型よりもプル型が望ましい](./recommendations#_8)ことのもうひとつの理由です。

未知のサイズの配列を絶対にループする必要がある場合は、複数のブロックを取る可能性があるため、複数のトランザクションが必要になる可能性があります。
次の例のように、あなたがどれぐらい離れているかを追跡し、その時点から再開できるようにする必要があります。

```sol
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```

`payOut()`関数の次の反復を待っている間に他のトランザクションが処理されても何の悪いことも起こらないようにする必要があります。
このパターンは、絶対に必要な場合にのみ使用してください。


### Gas Limit DoS on the Network via Block Stuffing

Even if your contract does not contain an unbounded loop, an attacker can prevent other transactions from being included in the blockchain for several blocks by placing computationally intensive transactions with a high enough gas price.

To do this, the attacker can issue several transactions which will consume the entire gas limit, with a high enough gas price to be included as soon as the next block is mined. No gas price can guarantee inclusion in the block, but the higher the price is, the higher is the chance.

If the attack succeeds, no other transactions will be included in the block. Sometimes, an attacker's goal is to block transactions to a specific contract prior to specific time.

This attack [was conducted](https://osolmaz.com/2018/10/18/anatomy-block-stuffing) on Fomo3D, a gambling app. The app was designed to reward the last address that purchased a "key". Each key purchase extended the timer, and the game ended once the timer went to 0. The attacker bought a key and then stuffed 13 blocks in a row until the timer was triggered and the payout was released. Transactions sent by attacker took 7.9 million gas on each block, so the gas limit allowed a few small "send" transactions (which take 21,000 gas each), but disallowed any calls to the `buyKey()` function (which costs 300,000+ gas).

A Block Stuffing attack can be used on any contract requiring an action within a certain time period. However, as with any attack, it is only profitable when the expected reward exceeds its cost. Cost of this attack is directly proportional to the number of blocks which need to be stuffed. If a large payout can be obtained by preventing actions from other participants, your contract will likely be targeted by such an attack. 

## Insufficient gas griefing

This attack may be possible on a contract which accepts generic data and uses it to make a call another contract (a 'sub-call') via the low level `address.call()` function, as is often the case with multisignature and transaction relayer contracts.

If the call fails, the contract has two options:

1. revert the whole transaction
2. continue execution.

Take the following example of a simplified `Relayer` contract which continues execution regardless of the outcome of the subcall:

```sol
contract Relayer {
    mapping (bytes => bool) executed;
    
    function relay(bytes _data) public {
        // replay protection; do not call the same transaction twice
        require(executed[_data] == 0, "Duplicate call");
        executed[_data] = true;
        innerContract.call(bytes4(keccak256("execute(bytes)")), _data);
    }
}
```

This contract allows transaction relaying. Someone who wants to make a transaction but can't execute it by himself (e.g. due to the lack of ether to pay for gas) can sign data that he wants to pass and transfer the data with his signature over any medium. A third party "forwarder" can then submit this transaction to the network on behalf of the user.

If given just the right amount of gas, the `Relayer` would complete execution recording the `_data`argument in the `executed` mapping, but the subcall would fail because it received insufficient gas to complete execution.

Note: When a contract makes a sub-call to another contract, the EVM limits the gas forwarded to [to 63/64 of the remaining gas](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md), 

An attacker can use this to censor transactions, causing them to fail by sending them with a low amount of gas. This attack is a form of "[griefing](https://en.wikipedia.org/wiki/Griefer)": It doesn't directly benefit the attacker, but causes grief for the victim. A dedicated attacker, willing to consistently spend a small amount of gas could theoretically censor all transactions this way, if they were the first to submit them to `Relayer`.

One way to address this is to implement logic requiring forwarders to provide enough gas to finish the subcall. If the miner tried to conduct the attack in this scenario, the `require` statement would fail and the inner call would revert. A user can specify a minimum gasLimit along with the other data (in this example, typically the `_gasLimit` value would be verified by a signature, but that is ommitted for simplicity in this case).

```sol
// contract called by Relayer
contract Executor {
    function execute(bytes _data, uint _gasLimit) {
        require(gasleft() >= _gasLimit);
        ...
    }
}
```

Another solution is to permit only trusted accounts to mine the transaction. 


## Forcibly Sending Ether to a Contract

Etherをコントラクトに強制的に送信することは、フォールバック機能をトリガーすることなく可能です。
これは、フォールバック機能に重要なロジックを配置したり、コントラクトの残高に基づいて計算を行う際に重要な考慮事項です。
次の例を考えてみましょう。

```sol
contract Vulnerable {
    function () payable {
        revert();
    }
    
    function somethingBad() {
        require(this.balance > 0);
        // Do something bad
    }
}
```

このロジックはコントラクトへの支払いを拒否し、何か悪いことが起こらないようにしているようです。
しかしながら、Etherを強制的にコントラクトに送り、それゆえ、そのバランスをゼロより大きくするためのいくつかの方法が存在します。

`selfdestruct`メソッドは、ユーザーが余分なEtherを送信する受益者を指定することを可能にします。(
[does not trigger a contract's fallback function](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether))

展開する前に、コントラクトのアドレスを[precompute](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo)に送信し、
そのアドレスにEtherを送信することもできます。

コントラクトエンジニアは、Etherを強制的に送ることができることに注意し、それに応じてロジックを設計する必要があります。
一般的に、資金調達源をコントラクトに制限することはできないと仮定します。

## Deprecated/historical attacks

これは、プロトコルの変更や強固性の向上により、もはや不可能な攻撃です。 後世のためにここに記録されています。

### Call Depth Attack (deprecated)

[EIP 150](https://github.com/ethereum/EIPs/issues/150)ハードフォークの時点では、Call Depth攻撃はもはや意味がありません。
<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup>
(すべてのガスは1024の深さ制限に達する前に十分消費されるでしょう)

## Other Vulnerabilities

The [Smart Contract Weakness Classification Registry](https://smartcontractsecurity.github.io/SWC-registry/) offers a complete and up-to-date catalogue of known smart contract vulnerabilities and anti-patterns along with real-world examples. Browsing the registry is a good way of keeping up-to-date with the latest attacks.
