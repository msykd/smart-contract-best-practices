多額の資金を必要とするコントラクトやミッションクリティカルなコントラクトが必要な場合は、適切なドキュメントを添付することが重要です。
関連するドキュメントには以下のものがあります。

## 仕様とロールアウトの計画

- 監査員、査読者、およびコミュニティが、システムが意図するものを理解するのに役立つ仕様、図、状態マシン、モデル、およびその他のドキュメンテーション。
- 多くのバグは仕様書から見つけることができ、修正するのが最もコストがかかりません。
- [ここに](https://github.com/ConsenSys/smart-contract-best-practices#contract-rollout)記載されている詳細、及び対象日付を含むロールアウトの計画。

## 情報

- 最新版のコードがデプロイされている場所。
- コンパイラのバージョン、使用されるフラグ、およびデプロイされたバイトコード。
- ロールアウトで使用されるコンパイラのバージョンとフラグ。
- デプロイされたコードの現状（未解決の問題、パフォーマンス統計などを含む）

## 既知の問題

- コントラクト上の主要リスク
  - 例えば全てのお金を失う、ハッカーが特定の結果に投票できる
- 既知のバグ、制限事項
- 潜在的な攻撃と緩和策
- 潜在的な利益相反 (例: will be using yourself, like Slock.it did with the DAO)

## 歴史

- テスト（使用統計、発見されたバグ、テストの長さを含む）
- コードをレビューした人(そして主要なフィードバック)

## 手順

- バグが発見された場合の行動計画(例:緊急のオプション、公開通知プロセスなど)
- 何かが間違っている場合、プロセスを終了する（たとえば、資金提供者は、残高から攻撃前に残高の割合を取得します）
- 責任ある開示方針（例えば、発見されたバグを報告する場所、バグバウンティプログラムのルール）
- 失敗の場合の援助（例えば、保険、ペナルティ・ファンド、償還不可）

## 問い合わせ先

- 問題が起こった時の問い合わせ先
- プログラマーやその他の重要な関係者の名前
- 質問が出来るチャットルーム