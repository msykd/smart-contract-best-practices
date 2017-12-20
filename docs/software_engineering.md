既知の攻撃を防ぐだけでは不十分です。
ブロックチェーン上の障害のコストは非常に高くなる可能性があるため、そのリスクを考慮に入れてソフトウェアの書き方に適用する必要があります。

私たちが提唱するアプローチは、「失敗に備える」ことです。
コードが安全かどうかを事前に知ることは不可能です。
ただしコントラクトを、失敗しても最小限の損害を与えるように設計することはできます。
このセクションでは、障害の準備に役立つさまざまなテクニックを紹介します。

注:システムに新しいコンポーネントを追加すると、常に危険が生じます。
誤って設計されたフェールセーフが脆弱性になる可能性があります。
堅牢なシステムを構築するために、コントラクトで使用するそれぞれのテクニックをどう連携するか注意深く検討してください。

### 壊れたコントラクトのアップグレード

エラーが発見された場合、または改善が必要な場合は、コードを変更する必要があります。 バグを発見するのは良いことではありませんが、対処する方法はありません。

スマートコントラクトのための効果的なアップグレードシステムを設計することは積極的な研究の領域であり、このドキュメントではすべての複雑な問題をカバーすることはできません。
しかし、最も一般的に使用される2つの基本的なアプローチがあります。
2つのうちの単純なものは、コントラクトの最新バージョンのアドレスを保持するレジストリコントラクトを持つことです。
ユーザーにとってよりシームレスなアプローチは、最新のコントラクトにデータを転送することです。

どんな技術であれ、コンポーネントのモジュール化と適切な分離が重要であり、コードの変更によって機能が損なわれたり、データが孤立したり、
移植に多額のコストがかかることはありません。特に、複雑なロジックをデータストレージから分離すると、機能を変更するためにすべてのデータを再作成する必要はありません。

また、当事者がコードをアップグレードすることを決定する安全な方法を持つことも重要です。
契約に応じて、コード変更は、単一の信頼できる当事者、メンバーのグループ、または一連のステークホルダーの投票によって承認される必要があります。
このプロセスに時間がかかる場合は、
[緊急停止またはサーキットブレーカー]（https://github.com/ConsenSys/smart-contract-best-practices/#circuit-breakers-pause-contract-functionality）など、
攻撃の際により迅速に対応する他の方法があるかどうかを検討する必要があります。

**例1: レジストリコントラクトを使用して最新バージョンのコントラクトを保存する**

この例では、callは転送されないため、ユーザーはcallする前に現在のアドレスを取得する必要があります。

```sol
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
    }

    function changeBackend(address newBackend) public
    onlyOwner()
    returns (bool)
    {
        if(newBackend != backendContract) {
            previousBackends.push(backendContract);
            backendContract = newBackend;
            return true;
        }

        return false;
    }
}
```

このアプローチには主に2つの欠点があります。

1.ユーザーは常に現在のアドレスを検索しなければなりません。そうしないと、古いバージョンのコントラクトを使用するリスクが発生します。
2.コントラクトを取り替えるときにデータをどのように扱うかについては、注意深く考える必要があります。

別のアプローチでは、最新のコントラクトにデータを転送する必要があります。

**例2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)を使用してデータを転送する**

```sol
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // this owner may be another contract with multisig, not a single contract owner
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        require(currentVersion.delegatecall(msg.data));
    }
}
```

このアプローチは、以前の問題を回避するが、それ自体の問題を有する。
このコントラクトにどのようにデータを格納するかについては、非常に注意する必要があります。
新規コントラクトのストレージレイアウトが最初のものと異なる場合、データが破損する可能性があります。
さらに、このシンプルなバージョンのパターンでは、関数から値を返すことはできません。
それらを転送するだけで、その適用性が制限されます。
（[もっと複雑な実装]（https://github.com/ownage-ltd/ether-router）は、インラインアセンブリコードと戻り値のレジストリでこれを解決しようとしています。）

あなたのアプローチにかかわらず、あなたのコントラクトをアップグレードするには何らかの方法があることが重要です。
そうでなければ、不可避のバグが発見されたときには使用できなくなります。

### サーキットブレーカー(コントラクトの一時停止)

サーキットブレーカーは、特定の条件が満たされると実行を停止し、新しいエラーが検出されたときに役立ちます。
たとえばバグが発見された場合、ほとんどのアクションは中断され、現在アクティブなアクションは取り消します。
特定の関係者にサーキットブレーカーをトリガーする機能を与えるか、
特定の条件が満たされたときに特定のブレーカーを自動的にトリガーするプログラムルールを与えることができます。

例:

```sol
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public {
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public {
    // some code
}

function withdraw() onlyInEmergency public {
    // some code
}
```

### スピードバンプ (遅延契約アクション)

スピードバンプはアクションを遅くするので、悪意のあるアクションが発生した場合、回復する時間があります。
例えば、[The DAO]（https://github.com/slockit/DAO/）は、DAOを分割する要求が成功してからその能力を得るまでに27日間必要でした。
これにより、資金が契約内に保たれ、回復の可能性が高められました。
DAOの場合、スピードバンプによって与えられた時間の間に取ることができる効果的なアクションはありませんでしたが、他のテクニックと組み合わせて、非常に効果的です。

Example:

```sol
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4 weeks

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // for simplicity, we withdraw everything;
        // presumably, the deposit function prevents new deposits when withdrawals are in progress

        requestedWithdrawals[msg.sender] = RequestedWithdrawal({
            amount: amountToWithdraw,
            time: now
        });
    }
}

function withdraw() public {
    if(requestedWithdrawals[msg.sender].amount > 0 && now > requestedWithdrawals[msg.sender].time + withdrawalWaitPeriod) {
        uint amountToWithdraw = requestedWithdrawals[msg.sender].amount;
        requestedWithdrawals[msg.sender].amount = 0;

        require(msg.sender.send(amountToWithdraw));
    }
}
```

### レート制限

レート制限は、実質的な変更を停止するか、または承認を必要とします。
例えば、預金者は一定期間（例えば、1日に最大100ether）の一定の預金額または一定割合の預金を引き出すことが許可されているだけで、
その期間の追加引き出しは失敗するか、何らかの特別な承認が必要となる 。
またはレート制限は、期間内に発行された一定量のトークンのみで、コントラクトレベルで行うことができます。

[例](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

### コントラクトロールアウト

資金が危険にさらされる前に、コントラクトには相当の長期間のテストが必要です。

最低限やるべきこと:

- 100%のテストカバレッジを持つ完全なテストスイートの作成
- 独自のテストネットに展開する
- テストとバグバウンティプログラムを組みパブリックなテストネットにデプロイ
- 徹底的なテストは、様々なプレイヤーがコントラクトとやりとりできるようにすべきである
- ベータ版でメインネット上に展開し、リスクの大きさ制限する

##### 自動廃止

テスト中は、特定の時間が経過した後で、アクションを防止して自動的に非推奨にすることができます。
たとえば、アルファコントラクトは数週間働いてから、最後の取り消しを除いてすべてのアクションを自動的に停止します。

```sol
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public isActive {
    // some code
}

function withdraw() public {
    // some code
}

```
##### ユーザー/コントラクト毎のEtherの量を制限する

初期段階では、あらゆるユーザー（またはコントラクト全体）のEtherの量を制限して、リスクを削減することができます。

### バグバウンティプログラム

バウンティプログラムを実行するためのヒント:

- どの通貨(BTC,ETHなど)の奨励金が分配されるかを決定する
- バウンティ報酬の推定総予算を決定する
- 予算から、3段階の報酬を決定します。
   - あなたが喜んで提供している最小の報酬
   - 通常は報酬の高い報酬
   - 非常に重大な脆弱性の場合に付与される余分な範囲
- 賞金の裁判官が誰であるかを決定する（3つは理想的かもしれない）
- 鉛の開発者はおそらく賞金の裁判官の1人であるべきです
- バグレポートが受信されると、主任開発者は、審査員の助言を得て、バグの重大度を評価する必要があります
- この段階での作業はプライベートレポにする必要があり、問題はGithub
- 修正する必要があるバグの場合、プライベートレポでは、開発者はテストケースを作成する必要があります。テストケースは失敗し、バグを確認する必要があります
- 開発者は修正プログラムを実装し、今すぐテストに合格する必要があります。必要に応じて追加のテストを書く
- バウンティハンターに修正を表示する。パブリック・リポジトリに修正をマージすることは1つの方法です
- バウンティハンターが修正に関する他のフィードバックを持っているかどうかを判断する
- 賞金裁判官は、バグの*可能性*と*影響*の両方の評価に基づいて、報酬のサイズを決定します。
- 賞金を受け取った参加者にその過程を通して情報を提供し、報酬を送ることの遅れを避けるために努力する

報酬の3段階の例については、[Ethereum's Bounty Program]（https://bounty.ethereum.org）を参照してください。

> 払い出される報酬の価値は、影響の重大さによって異なります。 マイナーな「無害」バグの報酬は0.05BTCから始まります。 例えばコンセンサスの問題につながる主要なバグは、最大で5 BTCまで報酬を受けるでしょう。 非常に重大な脆弱性の場合、より高い報酬（最大25 BTC）が可能です。