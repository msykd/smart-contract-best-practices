
# トークン実装のベストプラクティス

トークンを実装する際は、他のセキュリティベストプラクティスに準拠する必要がありますが、独自の考慮事項もあります。

## 最新の標準規格に準拠する

概して、トークンのスマートスマートコントラクトは、広く受け入れられている安定した標準規格に従うべきです。

現在受け入れられている標準規格の例は、

* [EIP20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
* [EIP721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) (non-fungibleトークン)

## EIP-20に対するフロントランニング攻撃に注意する

EIP-20トークンの `approve（）` 関数は、トークンの消費を承認をされた側が、意図した額を超える額を消費してしまう潜在的リスクをはらんでいます。
[フロントランニング攻撃](./known_attacks/#transaction-ordering-dependence-tod-front-running)を使用することで、トークンの消費を承認をされた側が `Approve（）` 呼び出しが処理される前と後の双方において `transferFrom（）` を呼び出すことができます。
詳細については、[EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve)および[このドキュメント](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit)で参照できます。

## 0x0アドレスへのトークン転送を禁止する

本稿執筆時点では、「ゼロ」アドレス([0x0000000000000000000000000000000000000000](https://etherscan.io/address/0x0000000000000000000000000000000000000000))は、80万ドルを超える値のトークンを保持しています。


## コントラクトアドレスへのトークン転送を禁止する

スマートコントラクトのアドレスへのトークンの転送もしないようにしてください。

この脆弱性をオープンのままにしておくことによる損失の可能性の例は、[EOS token smart contract](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0)で、9万トークン以上がコントラクトアドレスに溜まったままになってしまっています。


### 実装例

上記の推奨事項を実装する例として、次の修飾子を作成します。 "to"アドレスが0x0でもスマートコントラクト自身のアドレスでもないことを検証します。

```sol
    modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
```

修飾子は "transfer"と "transferFrom"メソッドに適用されるべきです：

```sol 
    function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool) 
    {
        (... your logic ...)
    }
```
