このページでは、スマートコントラクトを書くときに一般的に従うべきいくつかのパターンを示しています。

## プロトコル固有の推奨事項

### <!-- -->

以下の推奨事項は、Ethereum上のコントラクトシステムの開発に適用されます。

## 外部コール

### 外部コールを行う時は注意する

信頼されていないコントラクトを呼び出すと、いくつかの予期しないリスクやエラーが発生する可能性があります。
外部呼び出しは、そのコントラクトで悪意のあるコードを実行する可能性があります。
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

## 2者コントラクトまたはN者コントラクトでは、一部の参加者が「オフラインになる」可能性があることに注意する

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
例えば、トークン発行コントラクトにおけるトークンとetherの発行比率は固定されていてもよい。
これが常に `assert（）`の場合に当てはまることを確認することができます。
アサートガードは、コントラクトを一時停止し、アップグレードを許可するなど、他の手法と組み合わせることがよくあります。
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

アサーションは、コントラクトが `deposit（）`関数を経由せずに強制的にetherを送信することができるので、残高が厳密にイコールではないことに注意してください。

## `assert()` と `require()` プロパティを使う

Solidity 0.4.10から`assert()`と`require()`が導入されました。
`require（condition）`は入力の検証に使用されることを意味します。
これはユーザの入力に対して実行する必要があり、条件がfalseの場合に元に戻ります。
`assert（condition）`は、条件がfalseの場合でも元に戻りますが、内部エラーやコントラクトが無効な状態になったかどうかを確認するためにのみ使用してください。
このパラダイムに従うと、正式な分析ツールは、無効なオペコードに到達することができないことを検証することができます。
つまり、コード内のインバリアントが違反されていないことと、コードが正式に検証されたことを意味します。

## アサーションにのみ修飾子を使う

修飾子の内側のコードは通常関数本体の前に実行されるので、状態の変化や外部呼び出しは[Checks-Effects-Interactions](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern)パターンに違反します。
さらに、修飾子のコードは関数宣言からかけ離れているため、これらのステートメントも開発者には気付かれないままになることがあります。
たとえば、修飾子での外部呼び出しはリエントラント攻撃につながる可能性があります。

```sol
contract Registry {
    address owner;
    
    function isVoter(address _addr) external returns(bool) {
        // Code
    }
}

contract Election {
    Registry registry;
    
    modifier isEligible(address _addr) {
        require(registry.isVoter(_addr));
        _;
    }
    
    function vote() isEligible(msg.sender) public {
        // Code
    }
}
```

この場合、`Registry`コントラクトは`isVoter()`の中で`Election.vote()`を呼び出すことによってリエントラント攻撃を仕掛けることができます。

修飾子は[エラー処理](https://solidity.readthedocs.io/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions)にのみ使用してください。

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

## 抽象的なコントラクトとインタフェースの間のトレードオフに注意してください

インタフェースと抽象コントラクトの両方は、スマートコントラクトのためのカスタマイズ可能で再利用可能なアプローチを提供します。
Solidity 0.4.11で導入されたインタフェースは、抽象的なコントラクトに似ていますが、実装されている機能を持つことはできません。
インタフェースには、ストレージにアクセスできない、または抽象コントラクトをより実用的にする他のインタフェースから継承するなどの制限もあります。
しかし、インタフェースは実装前にコントラクトを設計する上では確かに有用です。
さらに、コントラクトが抽象コントラクトから継承する場合は、オーバーライドを使用して実装されていないすべての機能を実装する必要があること、または抽象的であることにも留意することが重要です。

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

## fallback関数でデータ長をチェックする

[fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)はEther転送のために呼び出されるだけでなく、他の関数が一致しないときにも呼び出されます。
そのため、fallback関数が受信されたEtherのログ記録の目的にのみ使用されることを意図している場合は、データが空であることを確認する必要があります。
そうでなければ、あなたのコントラクトが誤って使用されていて、存在しない機能が呼び出されても、呼び出し側は気付かないでしょう。

```sol
// bad
function() payable { LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
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
プラグマをロックすると、未知のバグのリスクが高い最新のコンパイラなどを使用して、コントラクトが誤って展開されないようになります。
コントラクトは他の人によっても展開される可能性があり、プラグマはオリジナルの著者が意図したコンパイラのバージョンを示します。

```sol
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

注：フローティングプラグマバージョン（例： `^0.4.25`）は`0.4.26-nightly.2018.9.25`で問題なくコンパイルできますが、本番用のコードをコンパイルするためにnightly buildを使用してはいけません。

### 例外

ライブラリまたはEthPMパッケージのコントラクトのように、コントラクトが他の開発者による使用を目的としている場合は、プラグマステートメントをフローティングさせることができます。そうでなければ、開発者はローカルにコンパイルするためにプラグマを手動で更新する必要があるでしょう。

## イベントを使用してコントラクトの活動を監視する

コントラクトのデプロイ後にコントラクトの活動を監視する方法があると便利です。これを実現する1つの方法は、コントラクトのすべてのトランザクションを調べることですが、コントラクト間のメッセージ呼び出しはブロックチェーンに記録されないため、これでは不十分な場合があります。さらに、入力パラメーターのみが表示され、実際の状態の変更は表示されません。

```sol
contract Charity {
    mapping(address => uint) balances;
    
    function donate() payable public {
        balances[msg.sender] += msg.value;
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}
```

ここでは、`Game`コントラクトは`Charity.donate()`への内部呼び出しを行います。このトランザクションは`Charity`のトランザクション一覧に表示されません。
たとえ`Game`のトランザクションを見ても、プレイヤーがコインを購入するために費やした金額のみが表示され、`Charity`コントラクトに使用された金額は表示されません。

イベントを介して両方の問題を解決することは可能です。イベントは、コントラクトで発生したことを記録するための便利な方法です。
発生したイベントは他のコントラクトデータとともにブロックチェーンに残り、将来の監査に使用できます。これは、チャリティの寄付の履歴を提供するためのイベントを使った、上記の例の改良です。

```sol
contract Charity {
    // define event
    event LogDonate(uint _amount);
    
    mapping(address => uint) balances;
    
    function donate() payable public {
        balances[msg.sender] += msg.value;
        // emit event
        emit LogDonate(msg.value);
    }
}

contract Game {
    function buyCoins() payable public {
        // 5% goes to charity
        charity.donate.value(msg.value / 20)();
    }
}

```

ここでは、直接またはそうでないかにかかわらず、`Charity`コントラクトを通過するすべての取引が寄付金額とともにそのコントラクトのイベントリストに表示されます。


## より新しいSolidity構造を用いる

`selfdestruct`（` suicide`）と `keccak256`（` sha3`以上）のような構造体/エイリアスが好ましいです。
`require（msg.sender.send（1 ether））`のようなパターンは `msg.sender.transfer（1 ether）`のように `transfer（）`を使って単純化することもできます。

## ビルトインはシャドーイング出来ることに注意する

Solidityの組み込みグローバルを[シャドーイング](https://en.wikipedia.org/wiki/Variable_shadowing)することは現在可能です。
これにより、コントラクトは `msg`や` revert() `などの組み込み関数の機能をオーバーライドすることができます。
これは[意図されてのことですが](https://github.com/ethereum/solidity/issues/1249)、コントラクトの真の振る舞いに関してユーザーを誤解させる可能性があります。

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

コントラクトユーザー（および監査人）は、使用するアプリケーションのスマートコントラクトのソースコード全体を知っておく必要があります。

## `tx.origin`を使わない

`tx.origin`を絶対に使用しないでください。
あなたのコントラクトにcallする方法（例えば、ユーザーに資金がある場合）と、
あなたのアドレスが`tx.origin`にあるようにトランザクションを承認する方法があります。

```sol
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
[Vitalik：'tx.originは引き続き使用可能または意味のあるものと仮定しないでください。](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200＃200)

`tx.origin`を使うコントラクトは他のコントラクトで使うことができないので、
コントラクト間の相互運用性を制限していることにも言及する価値があります。

## タイムスタンプ依存

タイムスタンプを使用してコントラクト内の重要な関数を実行する場合、特にアクションに資金移動が含まれる場合は、3つの主な考慮事項があります。

### タイムスタンプ操作

ブロックのタイムスタンプはマイナーによって操作される可能性があることに注意してください。こちらの[コントラクト](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code)を見てみましょう:

```sol
    
uint256 constant private salt =  block.timestamp;
    
function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y; 
    uint256 h = uint256(block.blockhash(seed)); 
    
    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

コントラクトがタイムスタンプを使用して乱数をシードする場合、マイナーはブロックが検証されてから15秒以内に実際にタイムスタンプをポストできます。これにより、マイナーは宝くじのチャンスに有利なオプションを事前計算することができます。タイムスタンプはランダムではないので、その場合は使用しないでください。

### 15秒ルール

[Yellow Paper](http://yellowpaper.io/) (Ethereum参照仕様)では、時間内にブロックがどれだけ漂う可能性があるかについての制約は規定されていません。
しかし、Yellow Paperでは各タイムスタンプがその親のタイムスタンプよりも大きくなければならないことを[規定しています](https://ethereum.stackexchange.com/a/5926/46821)。
一般的なEthereumプロトコルの実装である[Geth](https://github.com/ethereum/go-ethereum/blob/4e474c74dc2ac1d26b339c32064d0bac98775e77/consensus/ethash/consensus.go#L45)と[Parity](https://github.com/paritytech/parity-ethereum/blob/73db5dda8c0109bb6bc1392624875078f973be14/ethcore/src/verification/verification.rs#L296-L307)は、どちらも未来において15秒を超えたのタイムスタンプを持つブロックを拒否します。
> コントラクト関数がブロックが15秒間漂うことを許容できる場合は、`block.timestamp`を使用するのが安全です。

時間に依存するイベントの規模が15秒変化して整合性を維持できる場合は、タイムスタンプを使用しても安全です。

### タイムスタンプとして`block.number`を使用しない

`block.number`プロパティと[平均ブロック時間](https://etherscan.io/chart/blocktime)を使用して時間差を見積もることは可能ですが、ブロック時間が変わる可能性があるのでこれは将来にわたる証拠とはなりません（[fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/)や[difficulty bomb](https://github.com/ethereum/EIPs/issues/649)など）。
販売期間中は、15秒のルールにより、より信頼性の高い時間の見積もりを達成できます。


## 多重継承に注意

Solidityで多重継承を利用するときは、コンパイラが継承グラフをどのように構成するかを理解することが重要です。

```sol

contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;
    
    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;
    
    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```
コントラクトがデプロイされると、コンパイラは継承を右から左に*線形化*します（_is_ キーワードに続いて、親コントラクトは最も基底となるものから派生されているコントラクトまで、リスト化されます。）。
これがコントラクトAの線形化です。:

**Final <- B <- C <- A**

Cが最も派生したコントラクトであるため、線形化の結果、`fee` の値は5をもたらします。
これは明らかに思われるかもしれませんが、Cが重要な関数をシャドーイングし、ブール項を並べ替え、そして開発者に悪用可能なコントラクトを書かせることができてしまうシナリオを想像してください。
静的解析は現在、シャドーイング関数に関して問題を提起していないので、手動で検査しなければなりません。

セキュリティと継承の詳細については、この[記事](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)をチェックしてください。

コントリビューションの一助となるよう、SolidityのGithubはすべての継承関連の問題を含む[プロジェクト](https://github.com/ethereum/solidity/projects/9#card-8027020) を持っています。

## 型安全のためにアドレス型の代わりにインターフェース型を使用する

関数が引数としてコントラクトアドレスを取るとき、生の`address`ではなく、インターフェースまたはコントラクトタイプを渡すことをお勧めします。
この関数がソースコード内の別の場所で呼び出された場合、コンパイラは型の安全性をさらに保証します。

これには2つの選択肢があります。

```sol
contract Validator {
    function validate(uint) external returns(bool);
}

contract TypeSafeAuction {
    // good
    function validateBet(Validator _validator, uint _value) internal returns(bool) {
        bool valid = _validator.validate(_value);
        return valid;
    }
}

contract TypeUnsafeAuction {
    // bad
    function validateBet(address _addr, uint _value) internal returns(bool) {
        Validator validator = Validator(_addr);
        bool valid = validator.validate(_value);
        return valid;
    }
}
```

上記の`TypeSafeAuction`コントラクトを使用する利点は、次の例からわかります。
`validateBet()`が`address`引数、またはValidator以外のコントラクトタイプで呼び出された場合、コンパイラは次のエラーをスローします。

```sol
contract NonValidator{}

contract Auction is TypeSafeAuction {
    NonValidator nonValidator;
  
    function bet(uint _value) {
        bool valid = validateBet(nonValidator, _value); // TypeError: Invalid type for argument in function call.
                                                        // Invalid implicit conversion from contract NonValidator 
                                                        // to contract Validator requested.
    }
}
```

## 外部所有アカウントの確認に`extcodesize`を使用しない

次の修飾子（または同様のチェック）は、関数呼び出しが外部所有アカウント（EOA）または契約アカウントのどちらから行われたのかを確認するためによく使用されます。

```sol
// bad
modifier isNotContract(address _a) {
  uint size;
  assembly {
    size := extcodesize(_a)
  }
    require(size == 0);
     _;
}
```

考え方は簡単です。アドレスにコードが含まれている場合、それはEOAではなくコントラクトアカウントです。しかし、コントラクトでは構築中に利用可能なソースコードがありません。
これは、コンストラクタが実行されている間に、他のコントラクトを呼び出すことができますが、そのアドレスに対する`extcodesize`はゼロを返します。以下は、このチェックを回避する方法を示す最小限の例です。

```sol
contract OnlyForEOA {    
    uint public flag;
    
    // bad
    modifier isNotContract(address _a){
        uint len;
        assembly { len := extcodesize(_a) }
        require(len == 0);
        _;
    }
    
    function setFlag(uint i) public isNotContract(msg.sender){
        flag = i;
    }
}

contract FakeEOA {
    constructor(address _a) public {
        OnlyForEOA c = OnlyForEOA(_a);
        c.setFlag(1);
    }
}
```

コントラクトアドレスは事前計算できるため、ブロック`n`では空であるが、`n`より大きいあるブロックでコントラクトがデプロイされているアドレスをチェックする場合、このチェックも失敗する可能性があります。

!!! warning

    本項は微妙な問題となっています。

    他のコントラクトが自分のコントラクトにコールできないようにすることを目的としている場合は、おそらく`extcodesize`チェックで十分です。
    別の方法は`(tx.origin == msg.sender)`の値をチェックすることですが、これには[欠点](recommendations/#avoid-using-txorigin)もあります。

    `extcodesize`チェックがあなたの目的にかなう他の状況があるかもしれません。ここでそれらすべてを記述することは範囲外です。ここでそれらすべてを記述することは範囲外です。
    EVMの根本的な振る舞いを理解し、ご自身で判断をしてください。

    

## 廃止予定/過去の推奨事項

これらは、プロトコルの変更や強固性の改善によりもはや関連性のない推奨事項です。後世のためにここに記録されています。

### ゼロによる除算に注意 (Solidity < 0.4)

バージョン0.4以前のSolidityでは、数値をゼロで割ったときに例外を「throw」しません。 バージョン0.4以上で動作していることを確認してください。

### 関数とイベントを区別する (Solidity < 0.4.21)

[v0.4.21](https://github.com/ethereum/solidity/blob/develop/Changelog.md#0421-2018-03-07) において、Solidityは、`emit`キーワードを導入しました。以後は、`emit EventName();`のようにイベントを明示します。0.5.0以降は必須です。

関数とイベントの混同のリスクを防ぐために、大文字の使用とイベントの前の接頭辞（*Log*）を推奨します。関数の場合は、コンストラクタを除いて、常に小文字で始めます。

```sol
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```
