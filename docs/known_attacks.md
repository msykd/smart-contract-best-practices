以下は既知の攻撃方法の一覧です。スマートコントラクトを書く上でこれらの対策を行ってください。

## Race Conditions<sup><a href='#footnote-race-condition-terminology'>\*</a></sup>

外部コントラクトを呼び出す際の主要な危険の1つは、コントロールフローを引き継ぎ、呼び出し関数が期待していなかったデータを変更できることです。
このクラスのバグはいろいろな形を取ることができ、DAOの崩壊につながった主要なバグの両方がこの種のバグでした。

### Reentrancy

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

機能の最後までユーザーの残高が0に設定されていないため、2回目以降の呼び出しは引き続き成功し、何度も何度も残高を回収します。
非常によく似たバグが、The DAOの脆弱性の1つでした。

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

### Cross-function Race Conditions

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
この例では、両方の関数が同じ契約の一部であったことにも注意してください。
ただし、同じコントラクトが複数のコントラクトを共有している場合、複数のコントラクトで同じバグが発生する可能性があります。

### Pitfalls in Race Condition Solutions

競合状態は複数の機能、さらには複数のコントラクトにわたって発生する可能性があるため、再入を防止するためのソリューションは十分ではありません。

代わりに、最初にすべての内部作業を終了してから、外部関数を呼び出すことを推奨しました。
このルールは、慎重に従うと競争条件を避けることができます。
しかし、外部関数をあまりにも早く呼び出さないようにするだけでなく、外部関数を呼び出す関数を呼び出さないようにする必要があります。
たとえば、以下は安全ではありません。

```sol
// INSECURE
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    rewardsForA[recipient] += 100;
    withdraw(recipient); // At this point, the caller will be able to execute getFirstWithdrawalBonus again.
    claimedBonus[recipient] = true;
}
```

`getFirstWithdrawalBonus()`は外部コントラクトを直接呼び出さなくても、 `withdraw()`の呼び出しは競合状態に脆弱にするのに十分です。
したがって、 `withdraw()`も信頼できないように扱う必要があります。

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // Each recipient should only be able to claim the bonus once

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdraw(recipient); // claimedBonus has been set to true, so reentry is impossible
}
```

再入力が不可能であることに加えて、[信頼できない関数がマークされてます。](https://github.com/ConsenSys/smart-contract-best-practices#mark-untrusted-contracts)
この同じパターンは、すべてのレベルで繰り返されます。
`untrustedGetFirstWithdrawalBonus()`は外部コントラクトを呼び出す `untrustedWithdraw()`を呼び出すので、
`untrustedGetFirstWithdrawalBonus()`も安全でないものとして扱わなければなりません。

しばしば提案される別の解決策は、[mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)です。
これにより、ロックの所有者だけが変更できるように、状態を「ロック」することができます。 簡単な例は次のようになります。

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

ユーザーが最初の呼び出しが終了する前に `withdraw()`をもう一度呼び出そうとすると、ロックによって何も効果がありません。
これは効果的なパターンかもしれませんが、協力が必要な複数のコントラクトがある場合は面倒です。
以下は安全ではありません。

```sol
// INSECURE
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == 0);
        lockHolder = msg.sender;
    }

    function releaseLock() {
        lockHolder = 0;
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

攻撃者は `getLock()`を呼び出し、 `releaseLock()`を決して呼び出すことはできません。
彼らがこれをすると、契約は永遠にロックされ、それ以上の変更はできません。
競合状態を防ぐためにmutexを使用する場合は、ロックを要求して解放する方法がないことを慎重に確認する必要があります。
(デッドロックやライブロックのようなmutexを使ってプログラミングするときには、潜在的な危険性がありますので、mutexに書かれている大量の文献を参考にしてください。

<a name='footnote-race-condition-terminology'></a>

<div style='font-size: 80%; display: inline;'>* Some may object to the use of the term <i>race condition</i> since Ethereum does not currently have true parallelism. However, there is still the fundamental feature of logically distinct processes contending for resources, and the same sorts of pitfalls and potential solutions apply.</div>

## Transaction-Ordering Dependence (TOD) / Front Running

上記は攻撃者が単一のトランザクションで悪質なコードを実行することを含む競合状態の例です。
以下は、ブロックチェーンに固有の競合状態の異なるタイプです。つまり、トランザクションの順序(ブロック内)が操作の対象になりやすいという事実です。

しばらくの間、トランザクションはmempool内にあるので、ブロック内に入る前に、どのアクションが起こるかを知ることができます。
これは、分権化された市場のように、いくつかのトークンを購入するトランザクションを見ることができ、
他のトランザクションが含まれる前に実装された市場秩序のために厄介なことがあります。
特定のコントラクトそのものになるため、これを防ぐことは困難です。
たとえば、市場では、バッチオークションを実装する方がよいでしょう(これはまた、高頻度取引の懸念から保護します)。
事前コミットスキームを使用する別の方法もあります。後で詳細を記します。

## Timestamp Dependence

ブロックのタイムスタンプは、マイナーによって操作され、タイムスタンプの直接的および間接的な使用をすべて考慮する必要があることに注意してください。
*ブロック数*と*平均ブロック時間*は時間の見積もりに使用できますが、ブロック時間が変更される可能性がある(キャスパーで予想される変更など)将来の証拠ではありません。

```sol
uint someVariable = now + 1;

if (now % 2 == 0) { // the now can be manipulated by the miner

}

if ((someVariable - 100) % 2 == 0) { // someVariable can be manipulated by the miner

}
```

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

天びんが最大のuint値(2 ^ 256)に達すると、ゼロに丸く戻ります。これにより、その状態がチェックされます。
これは、実装に応じて適切かどうかは関係ありません。
uint値にこのような大きな値に近づく機会があるかどうかについて考えてみましょう。
uint変数がどのように状態を変え、そのような変更を行う権限を持っているかについて考えます。
uint値を更新する関数を呼び出すことができるユーザーがいる場合、攻撃を受けやすくなります。
管理者だけが変数の状態を変更するためのアクセス権を持っている場合、あなたは安全かもしれません。
ユーザーが一度に1だけインクリメントできる場合は、この制限に達する実現可能な方法がないため、おそらく安全です。

アンダーフローについても同様です。 uintが0より小さくなると、アンダーフローが発生し、最大値に設定されます。
uint8、uint16、uint24 ...などの小さなデータ型には注意してください。さらに簡単に最大値に達することができます。

[20 cases for overflow and underflow](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925).

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
前に説明したように、
[pull payment system](https://github.com/ConsenSys/smart-contract-best-practices/#favor-pull-over-push-payments)
を代わりに設定することをお勧めします。

もう1つの例は、契約によってユーザー(例えば、クラウドファンディングコントラクトの支援者)に支払うために配列を反復する場合である。
それぞれの支払いが成功することを確認することが一般的です。
そうでない場合は、元に戻す必要があります。 問題は、1つの呼び出しが失敗した場合、支払いシステム全体を元に戻すことです。
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

ここでも推奨の解決策は [favor pull over push payments](#favor-pull-over-push-payments)です。

## DoS with Block Gas Limit

あなたは前の例に別の問題があることに気づいたかもしれません。
一度に全員に払い戻すことで、ブロックガス制限にぶつかる危険性があります。
各Ethereumブロックは、ある最大量の計算を処理することができます。それを越えようとすると、あなたのトランザクションは失敗します。

これは、意図的な攻撃がなくても問題につながる可能性があります。
しかし、攻撃者が必要なガスの量を操作できるならば、特に悪いことです。
前の例の場合、攻撃者はアドレスの束を追加することができ、それぞれは非常に小さな払い戻しを必要とします。
したがって、各攻撃者のアドレスを払い戻すためのガスコストは、ガス制限を超えてしまい、払い戻し取引がまったく起こらないようになる可能性があります。

これが[favor pull over push payments](#favor-pull-over-push-payments)のもうひとつの理由です。

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

## Forcibly Sending Ether to a Contract

Etherをコントラクトに強制的に送信することは、フォールバック機能をトリガーすることなく可能です。
これは、フォールバック機能に重要なロジックを配置したり、契約の残高に基づいて計算を行う際に重要な考慮事項です。
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