このページでは、スマートコントラクトを書くときに一般的に従うべきいくつかのパターンを示しています。

## プロトコル固有の推奨事項

### <!-- -->

以下の推奨事項は、Ethereum上の契約システムの開発に適用されます。

## 外部コール

### 外部コールを行う時は注意する

信頼されていないコントラクトを呼び出すと、いくつかの予期しないリスクやエラーが発生する可能性があります。
外部呼び出しは、その契約で悪意のあるコードを実行する可能性があります。
したがって、すべての外部コールは潜在的なセキュリティリスクとして扱う必要があります。
外部コールを取り除くことができない、または望ましくない場合は、このセクションの残りのセクションの推奨事項を使用して危険を最小限に抑えてください。

### 信用できないコントラクトにマークをつける

外部コントラクトと対話するときは、変数、メソッド、およびコントラクト・インターフェースに、それらとの相互作用が潜在的に危険なものであることを明確にするような名前を付けます。
これは、外部コントラクトを呼び出す独自の関数に適用されます。

```sol
// bad
Bank.withdraw(100); // Unclear whether trusted or untrusted

function makeWithdrawal(uint amount) { // Isn't clear that this function is potentially unsafe
    Bank.withdraw(amount);
}

// good
UntrustedBank.withdraw(100); // untrusted external call
TrustedBank.withdraw(100); // external but trusted bank contract maintained by XYZ Corp

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```

### 外部コール後の状態変化は避ける

*raw calls* (`someAddress.call()`) や *contract calls* (`ExternalContract.someMethod()`)の
どちらを使用する場合でも、悪質なコードが実行される可能性があるとします。外部コントラクトが悪意のあるものではないとしても、それが呼び出すすべてのコントラクトによって悪質なコードが実行される可能性があります。

特に危険なのは、悪意のあるコードが制御フローを乗っ取ってリエントラント（再入可能性）による脆弱性を引き起こす可能性があることです。(この問題の詳細な議論については、[Reentrancy](https://github.com/ConsenSys/smart-contract-best-practices/known_attacks#reentrancy)を参照してください)

信頼できない外部コントラクトにコールをする場合は、*コール後の状態変化は避けてください*。このパターンは、[checks-effects-interactions pattern](http://solidity.readthedocs.io/en/develop/security-considerations.html?highlight=check%20effects#use-the-checks-effects-interactions-pattern)とも呼ばれます。


### `send()`, `transfer()`, `call.value()()`の間のトレードオフに注意する

etherを送信する方法として
`someAddress.send()`, `someAddress.transfer()`, `someAddress.call.value()()`などがあります。

- `someAddress.send()`と`someAddress.transfer()`は[リエントラント（再入可能性）](https://github.com/ConsenSys/smart-contract-best-practices/known_attacks#reentrancy)に対して安全と見なされます。
  これらのメソッドはコードを実行しますが、呼び出されるコントラクトには2,300ガスの義務のみが与えられ、現在はイベントを記録するのに十分です。
  
- `x.transfer(y)` は `require(x.send(y));`と同じで、送信が失敗すると元の状態に戻ります。
- `someAddress.call.value(y)()`は、提供されたetherとトリガーコードを送信します。
  実行されたコードには利用できるすべてのガスが与えられ、リエントラント（再入可能性）に対して安全ではありません。

`send()`や `transfer()`を使うとリエントラントを防ぐことができますが、フォールバック関数が2,300を超えるガスを必要とするコントラクトと互換性がないという犠牲を払って行います。
また、`someAddress.call.value(ethAmount).gas(gasAmount)()`を使ってカスタム量のガスを転送することもできます。

このトレードオフのバランスをとることを試みる1つのパターンは、[*プッシュ* と *プル*](https://consensys.github.io/smart-contract-best-practices/recommendations/#favor-pull-over-push-for-external-calls)の両方のメカニズムを実装することです。
*プッシュ*コンポーネントには `send()` または `transfer()` を使用し、*プル*コンポーネントには `call.value()()` を使用します。

valueの送信に `send()`や `transfer()` を排他的に使用しても、リエントラントに対しては安全ではなく、
特定のvalueを再入可能で安全にするということだけを指摘しておきましょう。

### 外部呼び出しのエラー処理

Solidityは、 `address.call()`、 `address.callcode()`、 `address.delegatecall()`、 `address.send()`のような生のアドレスで動作する低レベル呼び出しメソッドを提供します。
これらの低レベルメソッドは決して例外をスローしませんが、呼び出しが例外を検出した場合は `false`を返します。
一方、 `ExternalContract.doSomething()`などのコントラクト呼び出しは `doSomething()`がスローされると自動的にスローを伝播します(例えば、 `ExternalContract.doSomething()`も `throw`)。

低レベルのコールメソッドを使用する場合は、戻り値をチェックしてコールが失敗する可能性を処理してください。

```sol
// bad
someAddress.send(55);
someAddress.call.value(55)(); // this is doubly dangerous, as it will forward all remaining gas and doesn't check for result
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // if deposit throws an exception, the raw call() will only return false and transaction will NOT be reverted

// good
if(!someAddress.send(55)) {
    // Some failure code
}

ExternalContract(someAddress).deposit.value(100);
```


### 外部呼び出しでは*プッシュ*よりも*プル*型が望ましい 

外部呼び出しが誤ってまたは意図的に失敗する可能性があります。
このような障害によって引き起こされる被害を最小限に抑えるには、各外部呼び出しを、受信者が開始できる独自のトランザクションに分離する方がよい場合があります。
これは、特に支払いに関連します。ユーザーへ自動的に資金を送金（プッシュ）するのではなく、資金を引き出してもらう（プルしてもらう）ことをお勧めします。
(これにより、[ガスリミット問題](https://github.com/ConsenSys/smart-contract-best-practices/#dos-with-block-gas-limit)の可能性も減ります。)
単一のトランザクションで複数の`send()`呼び出しを組み合わせることは避けてください。

```sol
// bad
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            highestBidder.transfer(highestBid); // if this call consistently fails, no one else can bid
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// good
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != address(0)) {
            refunds[highestBidder] += highestBid; // record the refund that this user can claim
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        msg.sender.transfer(refund);
    }
}
```

### 信頼できないコードにdelegatecallをしない

`delegatecall` 関数は、あたかも呼び出し側のコントラクトに属しているかのように、他のコントラクトから関数を呼び出すために使用されます。
したがって、呼び出された側は状態を変更されてしまう可能性があります。これは安全ではないかもしれません。以下の例は、`delegatecall` を使用すると、コントラクトが破棄されて残高が失われる可能性があることを示しています。

```sol
contract Destructor
{
    function doWork() external
    {
        selfdestruct(0);
    }
}

contract Worker
{
    function doWork(address _internalWorker) public
    {
        // unsafe
        _internalWorker.delegatecall(bytes4(keccak256("doWork()")));
    }
}
```

デプロイされた`Destructor`コントラクトのアドレスを引数として`Worker.doWork()`が呼び出されると、Workerコントラクトはself-destructします。実行を信頼できるコントラクトにのみ委任し、ユーザーが指定したアドレスには委任しないでください。

## 新たにデプロイされたコントラクトの残高が0と決めつけない

デプロイ前にコントラクトアドレスに対してweiを送信できます。
コントラクトは初期状態に残高0が含まれていると想定すべきではありません。
詳細は[issue 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61)を参照してください。

## オンチェーンのデータは公開されていることを忘れない

多くのアプリケーションでは、データをある時点まで非公開にする必要があります。
ゲーム(例えば、チェーン上の岩石 - はさみ)とオークションメカニズム(例えば、入札価格の第二価格オークション)は、2つの主要なカテゴリの例です。
プライバシーが問題となるアプリケーションを構築する場合は、ユーザーに情報を早期に公開しないように注意してください。

例:

* じゃんけんでは、両方のプレイヤーに、最初に意図した移動のハッシュを提出する必要があります。送信された移動がハッシュと一致しない場合、例外をthrowします。
* オークションでは、プレイヤーに初期段階で入札額のハッシュ値を提示し(入札額を超える入金額とともに)、第2段階でアクション入札額を提示する必要があります。
* 乱数ジェネレータに依存するアプリケーションを開発する場合は、(1)プレイヤーが移動を提出する、(2)乱数が生成される、(3)プレイヤーが支払う順序が常に必要です。乱数が生成される方法自体は、活発な研究の領域です。現在のクラス最高のソリューションには、Bitcoinブロックヘッダー(http://btcrelay.org で検証済み)、ハッシュコミット公開スキーム(すなわち、一方の当事者が数値を生成し、そのハッシュを公開して値にコミットし、 後でその値を明らかにする)と[RANDAO](http://github.com/randao/randao)。
* 頻繁なバッチオークションを実装する場合、ハッシュコミットスキームも望ましいです。

## 2者契約またはN者契約では、一部の参加者が「オフラインになる」可能性があることに注意する

特定の要求を行っている特定の当事者に応じて払い戻しをしたり、他の方法で資金を引き出すことはできません。 
例えば、じゃんけんでは、共通の間違いの1つは、両方のプレーヤーが自分の動きを決定するまで支払いを行わないことです。
しかし、悪意のあるプレイヤーは、決して移動を提出しないことによって他のプレイヤーを「悲しませる」ことができます。
実際に、プレイヤーが他のプレイヤーの明らかな動きを見て、失ったと判断した場合、この問題はステートの決済チャネルのコンテキストでも発生する可能性があります。
そのような状況が問題である場合、(1)おそらく期限を過ぎて参加していない参加者を迂回させる方法を提供する、(2)参加者が参加しているすべての状況で情報を提出するための追加の経済的インセンティブを加えることを検討するそうするはずです。

## 最大の負の数の符号付き整数否定に注意する

Solidityは符号付き整数を扱うためのいくつかの型を提供します。ほとんどのプログラミング言語のように、Solidityでは`N`ビットの符号付き整数は`-2^(N-1)`から`2^(N-1)-1`までの値を表すことができます。これは、`MIN_INT`に正の等価物がないことを意味します。
否定は、2つの補数を見つけることで実装されます。 そのため、最も負の数の否定は[同じ数になります](https://en.wikipedia.org/wiki/Two%27s_complement#Most_negative_number)。

これはSolidityの全ての符号付き整数型（`int8`, `int16`, ..., `int256`）に当てはまります。

```sol
contract Negation {
    function negate8(int8 _i) public pure returns(int8) {
        return -_i;
    }
    
    function negate16(int16 _i) public pure returns(int16) {
        return -_i;
    }
    
    int8 public a = negate8(-128); // -128
    int16 public b = negate16(-128); // 128
    int16 public c = negate16(-32768); // -32768
}
```

これを処理する1つの方法は、否定の前に変数の値をチェックし、それが`MIN_INT`と等しい場合に例外をスローすることです。
もう1つの選択肢は、最大の負の数が、より大きな容量を持つ型（たとえば、`int16`ではなく`int32`）を使用して達成されないようにすることです。

`int`型に関する同様の問題は、`MIN_INT`が`-1`で乗算または除算されたときにも発生します。

## Solidity特有の推奨事項

### <!-- -->

以下の推奨事項は、Solidity固有のものですが、他の言語でスマートコントラクトを作成するための参考にもなります。

## `assert()`で不変量を強制する

アサーションが失敗した場合（不変プロパティの変更など）、アサートガードがトリガーされます。
例えば、トークン発行契約におけるトークンとetherの発行比率は固定されていてもよい。
これが常に `assert（）`の場合に当てはまることを確認することができます。
アサートガードは、契約を一時停止し、アップグレードを許可するなど、他の手法と組み合わせることがよくあります。
（でなければあなたはいつも失敗しているアサーションで立ち往生するかもしれません。）

例:

```sol
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(this.balance >= totalSupply);
    }
}
```

アサーションは、契約が `deposit（）`関数を経由せずに強制的にetherを送信することができるので、残高が厳密にイコールではないことに注意してください。

## `assert()` と `require()` プロパティを使う (Solidity >= 0.4.10)

Solidity 0.4.10から`assert()`と`require()`が導入されました。
`require（condition）`は入力の検証に使用されることを意味します。
これはユーザの入力に対して実行する必要があり、条件がfalseの場合に元に戻ります。
`assert（condition）`は、条件がfalseの場合でも元に戻りますが、内部エラーや契約が無効な状態になったかどうかを確認するためにのみ使用してください。
このパラダイムに従うと、正式な分析ツールは、無効なオペコードに到達することができないことを検証することができます。
つまり、コード内のインバリアントが違反されていないことと、コードが正式に検証されたことを意味します。

## 整数除算での丸めに注意する

すべての整数の除算は、最も近い整数に切り下げられます。さらに精度が必要な場合は、乗数を使用するか、分子と分母の両方を格納することを検討してください。

(将来的には、Solidityは固定小数点型を使用するため、これは簡単になります)

```sol
// bad
uint x = 5 / 2; // 結果は2で、すべての整数除算がDOWNから最も近い整数に丸められます。
```

乗数を使用すると四捨五入を防ぐことができます。この乗数は、将来xで作業するときに考慮する必要があります。

```sol
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

分子と分母を格納することは、 `分子/分母`の結果をオフチェーンで計算できることを意味します:
```sol
// good
uint numerator = 5;
uint denominator = 2;
```

## Etherは強制的にアカウントに送ることが出来る

厳密にコントラクトの残高をチェックする不変量をコーディングすることに注意してください。

攻撃者は任意のアカウントにweiを強制的に送ることができ、これは防ぐことができません（ `revert（）`を実行するフォールバック関数でさえも）。

攻撃者はコントラクトを作成し、1weiで資金を調達し、 `selfdestruct（victimAddress）`を呼び出すことでこれを行うことができます。
`victimAddress`ではコードが呼び出されないので、防ぐことはできません。

## 抽象的な契約とインタフェースの間のトレードオフに注意してください

インタフェースと抽象契約の両方は、スマートコントラクトのためのカスタマイズ可能で再利用可能なアプローチを提供します。
Solidity 0.4.11で導入されたインタフェースは、抽象的な契約に似ていますが、実装されている機能を持つことはできません。
インタフェースには、ストレージにアクセスできない、または抽象コントラクトをより実用的にする他のインタフェースから継承するなどの制限もあります。
しかし、インタフェースは実装前に契約を設計する上では確かに有用です。
さらに、契約が抽象コントラクトから継承する場合は、オーバーライドを使用して実装されていないすべての機能を実装する必要があること、または抽象的であることにも留意することが重要です。

## fallback関数をシンプルに保つ

[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)は、引数に引数なしのメッセージが送信されたとき（または関数が一致しないとき）に呼び出され、 `.send（）`または `.transfer（）`から呼び出されたとき2,300個のガスにアクセスできます。
あなたが `.send（）`や `.transfer（）`からEtherを受信できるようにしたいのであれば、fallback関数でできるのはイベントを記録することです。
計算以上のガスが必要な場合は、適切な関数を使用してください。

```sol
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { LogDepositReceived(msg.sender); }
```

## 関数と状態変数の可視性を明示的にマークする

関数と状態変数の可視性を明示的にラベル付けする。
関数は `external`、` public`、 `internal`、` private`のように指定できます。
それらの違いを理解してください。たとえば、 `public`ではなく` external`で十分でしょう。
状態変数については、「外部」は不可能である。
可視性を明示的にラベルすると、関数を呼び出すことができるか、変数にアクセスできるかについての間違った前提を簡単にキャッチできます。

```sol
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

## プラグマを特定のコンパイラのバージョンにロックする

コントラクトは、最もよくテストされたものと同じコンパイラ・バージョンとフラグでデプロイする必要があります。
プラグマをロックすると、未知のバグのリスクが高い最新のコンパイラなどを使用して、契約が誤って展開されないようになります。
コントラクトは他の人によっても展開される可能性があり、プラグマはオリジナルの著者が意図したコンパイラのバージョンを示します。

```sol
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

Note: a floating pragma version (ie. `^0.4.25`) will compile fine with `0.4.26-nightly.2018.9.25`, however nightly builds should never be used to compile code for production.

### Exception

Pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the developer would need to manually update the pragma in order to compile locally.

## 関数とイベントの命名規則

関数とイベントの混乱の危険を避けるため、イベント名は大文字で始め、大文字の前に接頭辞を付ける（*Log*を推奨する）。
関数の場合はコンストラクタを除き、常に小文字で始めます。

```sol
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

## より新しいSolidity構造を用いる

`selfdestruct`（` suicide`）と `keccak256`（` sha3`以上）のような構造体/エイリアスが好ましいです。
`require（msg.sender.send（1 ether））`のようなパターンは `msg.sender.transfer（1 ether）`のように `transfer（）`を使って単純化することもできます。

## ビルトインはシャドーイング出来ることに注意する

Solidityの組み込みグローバルを[shadow]（https://en.wikipedia.org/wiki/Variable_shadowing）することは現在可能です。
これにより、契約は `msg`や` revert（） `などの組み込み関数の機能をオーバーライドすることができます。
これは意図されていますが（https://github.com/ethereum/solidity/issues/1249）、コントラクトの真の振る舞いに関してユーザーを誤解させる可能性があります。

```sol
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

ユーザーは、使用するアプリケーションのコントラクトコードを認識している必要があります。

## `tx.origin`を使わない

`tx.origin`を絶対に使用しないでください。
あなたのコントラクトにcallする方法（例えば、ユーザーに資金がある場合）と、
あなたのアドレスが`tx.origin`にあるようにトランザクションを承認する方法があります。

```
pragma solidity 0.4.18;

contract MyContract {

    address owner;
    
    function MyContract() public {
        owner = msg.sender;
    }
    
    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        receiver.transfer(amount);
    }
    
}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }
    
}
```

承認のために `msg.sender`を使用するべきです
（別のコントラクトがあなたのコントラクトを呼び出す場合、` msg.sender`はコントラクトのアドレスであり、ユーザーのアドレスではありません）。

詳細はこちら: [Solidity docs](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

また、将来 `tx.origin` がEthereumプロトコルから削除される可能性があるので、 `tx.origin`を使用するコードは将来のリリースと互換性がありません。
[Vitalik：'tx.originは引き続き使用可能または意味のあるものと仮定しないでください。]（https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200＃200）

`tx.origin`を使うコントラクトは他のコントラクトで使うことができないので、
コントラクト間の相互運用性を制限していることにも言及する価値があります。

## 廃止予定/過去の推奨事項

これらは、プロトコルの変更や強固性の改善によりもはや関連性のない推奨事項です。後世のためにここに記録されています。

### ゼロによる除算に注意 (Solidity < 0.4)

バージョン0.4以前のSolidityでは、数値をゼロで割ったときに例外を「throw」しません。 バージョン0.4以上で動作していることを確認してください。
