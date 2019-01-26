### セキュリティ関連EIP

EVMがどのように機能するかを理解するためであったり、スマートコントラクトシステムを開発する際に適切なベストプラクティスを教示してくれたりする点において、次のEIPを踏まえておくことは大切なことです。

ただし、下記が網羅しているリストであるとは思わないでください。

### Final（正式採用）

- [EIP 155](https://eips.ethereum.org/EIPS/eip-155) Simple replay attack protection - Ethereum Classicなどの他のチェーン、さまざまなテストネット、またはコンソーシアムチェーンを操作せずに、Ethereumメインネットで機能するトランザクションを送信する方法を提供します。
- [EIP 214](https://eips.ethereum.org/EIPS/eip-214) コントラクトを呼び出している間の状態の変更を禁止しつつ、別のコントラクト（またはそれ自体）を呼び出すために使用できる新しいオペコードを追加します。
- [EIP 607](https://eips.ethereum.org/EIPS/eip-607) Hardfork Meta: Spurious Dragon - EIP 155からSimple Replay Attack Protectionを実装しました。
- [EIP 779](https://eips.ethereum.org/EIPS/eip-779) Hardfork Meta: DAO Fork - 「DAO Fork」という名前のハードフォークに含まれる変更を文書化しています。他のハードフォークとは異なり、DAO Forkはプロトコルを変更しませんでした。
というのは、DAOフォークは、アカウント（「child DAO」コントラクト）のether残高を特定のアカウント（「WithdrawDAO」コントラクト）に移す「イレギュラーな状態変更」でした。

### Draft（検討段階）

- [EIP 1470](https://eips.ethereum.org/EIPS/eip-1470) Smart Contract Weakness Classification - Ethereumスマートコントラクトのセキュリティ上の弱点に対する分類スキームを提案します（SWC）
- [EIP 1051](https://eips.ethereum.org/EIPS/eip-1051) Overflow checking for the EVM - 効率的な検出とオーバーフローの防止を可能にする2つの新しいオペコードを追加します。
- [EIP 1271](https://eips.ethereum.org/EIPS/eip-1271) Standard Signature Validation Method for Contracts - 多くのスマートコントラクトの現在の設計では、
コントラクトがプライベートキーを持たず、したがってメッセージに直接署名することができないため、コントラクトアカウントがスマートコントラクトとやり取りすることができません。
ここでの提案は、アカウントがコントラクトであるときに提供された署名が有効であるかどうかをコントラクトが検証するための標準的な方法の概要を示しています。
