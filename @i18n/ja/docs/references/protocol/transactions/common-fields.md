---
html: transaction-common-fields.html
parent: transaction-formats.html
seo:
    description: どのトランザクションについても、共通する一連のフィールドがあります。
labels:
  - トランザクション送信
---
# トランザクションの共通フィールド

どのトランザクションについても、共通する一連のフィールドに加え、[トランザクションのタイプ](types/index.md)に応じた追加のフィールドがあります。フィールドの名前では、大文字と小文字が区別されます。すべてのトランザクションに共通するフィールドは、以下のとおりです。

| フィールド              | JSONの型        | [内部の型][] | 説明      |
|:-------------------|:-----------------|:------------------|:-----------------|
| Account            | 文字列           | AccountID          | _（必須）_ トランザクションを開始した[アカウント](../../../concepts/accounts/index.md)の一意アドレス。 |
| TransactionType    | 文字列           | UInt16            | _（必須）_ トランザクションのタイプ。有効なタイプは、`Payment`、`OfferCreate`、`OfferCancel`、`TrustSet`、`AccountSet`、`SetRegularKey`、`SignerListSet`、`EscrowCreate`、`EscrowFinish`、`EscrowCancel`、`PaymentChannelCreate`、`PaymentChannelFund`、`PaymentChannelClaim`、`DepositPreauth`です。 |
| Fee                | 文字列           | Amount            | _（必須。[自動入力可能][]）_ 整数で表したXRPの額（drop単位）。このトランザクションをネットワークに送信するためのコストとして消却されます。トランザクションのタイプによっては、最小要件が異なります。詳細は、[トランザクションコスト][]をご覧ください。 |
| Sequence           | 符号なし整数 | UInt32            | _（必須。[自動入力可能][]）_ トランザクションを開始したアカウントに関連付けられた、トランザクションのシーケンス番号。トランザクションが有効とみなされるのは、その`Sequence`番号が、同一のアカウントの直前トランザクションよりも1大きい場合のみです。保留中のトランザクションを`Sequence`番号を使用して無効にする方法については、[トランザクションのキャンセルまたはスキップ](../../../concepts/transactions/finality-of-results/canceling-a-transaction.md)をご覧ください。 |
| [AccountTxnID][]   | 文字列           | Hash256           | _（省略可）_ 別のトランザクションを識別するためのハッシュ値。このハッシュがある場合、このトランザクションが有効になるのは、送信側のアカウントの直前送信トランザクションがこのハッシュと一致しているときのみです。 |
| [Flags][]          | 符号なし整数 | UInt32            | _（省略可）_ このトランザクションのビットフラグのセット。 |
| LastLedgerSequence | 数値           | UInt32            | _（省略可。使用を強く推奨）_ このトランザクションを登録できるレジャーインデックスの最大値。このフィールドを指定することにより、トランザクションが検証または拒否されるのを待たなければならない期間の上限を設定することができます。詳細は、[信頼できるトランザクションの送信](../../../concepts/transactions/reliable-transaction-submission.md)をご覧ください。 |
| [`NetworkID`](#networkidフィールド) | Number | UInt32           | _(Network-specific)_ The network ID of the chain this transaction is intended for. **MUST BE OMITTED** for Mainnet and some test networks. **REQUIRED** on chains whose network ID is 1025 or higher. |
| [Memos][]          | オブジェクトの配列 | 配列             | _（省略可）_ このトランザクションの識別に使用される任意の追加情報。 |
| [Signers][]        | 配列            | 配列             | _（省略可）_ このトランザクションを承認するための[マルチシグ](../../../concepts/accounts/multi-signing.md)を表すオブジェクトの配列。 |
| SourceTag          | 符号なし整数 | UInt32            | _（省略可）_ この支払いの理由、またはこのトランザクションの実行元である送信者を識別するために使用される任意の整数。一般的に、返金については、最初の支払いの`SourceTag`を返金の`DestinationTag`として指定する必要があります。 |
| SigningPubKey      | 文字列           | Blob              | _（署名時に自動追加）_ このトランザクションへの署名に使用される秘密鍵に対応する公開鍵の16進表現。空文字列の場合は、代わりに`Signers`フィールドにマルチシグが保持されていることを示します。 |
| TxnSignature       | 文字列           | Blob              | _（署名時に自動追加）_ このトランザクションが、発信元であると主張しているアカウントから発信されたものであることを検証するための署名。 |

[自動入力可能]: #自動入力可能なフィールド
[AccountTxnID]: #accounttxnid
[Flags]: #flagsフィールド
[Memos]: #memosフィールド
[Signers]: #signersフィールド

{% badge href="https://github.com/XRPLF/rippled/releases/tag/0.28.0" %}削除: rippled 0.28.0{% /badge %}: トランザクションの`PreviousTxnID`フィールドは、[AccountTxnID][]フィールドに置き換えられました。この文字列/Hash256フィールドは、過去に発生したトランザクションの一部に記述されています。このフィールドは、一部の[レジャーオブジェクト](../ledger-data/index.md)にある`PreviousTxnID`という同じ名前のフィールドとは無関係です。


## AccountTxnID

`AccountTxnID`フィールドにより、直前のトランザクション（シーケンス番号で識別）も有効で、かつ期待するトランザクションに一致しない限り、現在のトランザクションが有効にならないよう、トランザクションどうしをチェーンにすることができます。

このフィールドが有用になるのは、例えば、トランザクション送信用のプライマリーシステムと受動的なバックアップシステムを運用している場合です。受動的なバックアップシステムがプライマーリから切断されたものの、プライマリが完全に稼働停止となったわけではなく、両システムが同時に稼働を開始した場合は、トランザクションが2回送信される、あるいはまったく送信されないなど、深刻な問題が発生するおそれがあります。`AccountTxnID`を使用してトランザクションどうしをチェーンにすると、両方のシステムがアクティブになったときも、有効なトランザクションを送信できるのはいずれか一方のみとなります。

AccountTxnIDを使用するには、アカウントの1つ前のトランザクションのIDがレジャーで追跡されるよう、最初に[asfAccountTxnID](types/accountset.md#accountsetのフラグ)フラグを設定する必要があります。


## 自動入力可能なフィールド <a id="auto-fillable-fields"></a>

一部のフィールドについては、トランザクションの署名前に、`rippled`サーバによって、または署名に使用される[ripple-lib][]などのライブラリーによって値を自動入力できます。値を自動入力するには、最新の状態を取得するためのXRP Ledgerへのアクティブな接続が必要です。したがって、オフラインでは実行できません。[ripple-lib][]と`rippled`のどちらも、以下の値を自動的に提供できます。

* `Fee` - ネットワークに基づいて[トランザクションコスト][]を自動的に入力します。

    **注記:**`rippled`の[signメソッド][]を使用するときは、`fee_mult_max`パラメーターと`fee_div_max`パラメーターを使用して、自動入力値の上限を設定できます。

* `Sequence` - トランザクションを送信する側のアカウントの次のシーケンス番号を自動的に使用します。

本番システムについては、これらのフィールドの値がサーバによって入力される状態に _しない_ ことをお勧めします。例えば、ネットワークの負荷が一時的に急上昇したためにトランザクションコストが高騰した場合、トランザクションによっては、一時的な高額のコストを支払うよりも、必要に応じて待機し、コストが低下してから送信したほうが好ましいことがあります。

[Paymentトランザクション][]タイプの[`Paths`フィールド](types/payment.md#パス)についても、値を自動入力できます。


## Flagsフィールド

`Flags`フィールドには、トランザクションの行動を調整する各種のオプションを設定できます。オプションは、ビット単位のOR操作と組み合わせることで複数のフラグを同時に設定できるバイナリー値として表現します。

トランザクションで所定のフラグが有効になっているかどうかを確認するには、ビット単位のAND演算子をフラグの値と`Flags`フィールドで使用します。結果が0の場合は無効になっていることを示し、結果がフラグ値と等しい場合は有効になっていることを示します（その他の結果の場合は、実行した操作に誤りがあることを示します）。

ほとんどのフラグは、特定のタイプのトランザクションに対してのみ効果があります。複数のタイプのトランザクションに対して、同一のビット単位値をフラグに再利用できるため、フラグの設定と読み取りでは`TransactionType`フィールドに留意することが重要です。

フラグとして定義しないビットは、0にする必要があります([fix1543 Amendment][]では、一部のタイプのトランザクションについて、このルールが適用されます。デフォルトでは、ほとんどのタイプのトランザクションでこのルールが強制されます)。

### グローバルフラグ

すべてのトランザクションにグローバルに適用される唯一のフラグは、以下のとおりです。

| フラグの名前           | 16進値  | 10進値 | 説明               |
|:--------------------|:-----------|:--------------|:--------------------------|
| tfFullyCanonicalSig | 0x80000000 | 2147483648    |  _（使用を強く推奨）_ 完全に正規である署名を要求します。 |

[signメソッド][]（または「署名と送信」モードの[submitメソッド][]）を使用すると、`rippled`は、`Flags`フィールドがすでに存在している場合を除き、`tfFullyCanonicalSig`フラグを有効にした状態で`Flags`フィールドを追加します。`tfFullyCanonicalSig`フラグは、`Flags`が明示的に指定されている場合、自動的には有効に***なりません***。また、[sign_forメソッド][]を使用してマルチシグトランザクションに署名を追加する場合も、自動的には有効に***なりません***。

**警告:** `tfFullyCanonicalSig`を有効にしない場合は、不正使用者がトランザクションの署名を改変して、期待されるものとは別のハッシュを使用してトランザクションを成功させることが理論上可能になります。最悪の場合、同一の支払を何回も送信するようシステムに仕掛けられるおそれがあります。この問題を回避するには、署名するすべてのトランザクションで`tfFullyCanonicalSig`フラグを有効にします。

### フラグの範囲

トランザクションの`Flags`フィールドでは、さまざまなレベルや状況に適用されるフラグを設定できます。個々の状況に関するフラグは、以下の範囲に限定されます。

| 範囲の名前       | ビットマスク     | 説明                                |
|:-----------------|:-------------|:-------------------------------------------|
| ユニバーサルフラグ  | `0xff000000` | すべてのタイプのトランザクションに対して一様に適用されるフラグ。 |
| タイプに基づくフラグ | `0x00ff0000` | フラグを使用する[トランザクションのタイプ](types/index.md)に応じて意味が異なるフラグ。 |
| 予約済みのフラグ   | `0x0000ffff` | 現時点では定義されていないフラグ。トランザクションが有効になるのは、これらのフラグが無効になっている場合のみです。 |

**注記:** [AccountSetトランザクション][]タイプには、タイプに基づくフラグと似た目的を果たす[ビット単位ではない独自のフラグ](types/accountset.md#accountsetのフラグ)があります。[レジャーオブジェクト](../ledger-data/ledger-entry-types/index.md)にも、さまざまなビット単位のフラグが定義される`Flags`フィールドがあります。


## Memosフィールド

`Memos`フィールドは、トランザクションに関する任意のメッセージデータを保持します。このフィールドは、オブジェクトの配列として表現します。各オブジェクトには唯一のフィールド`Memo`があり、このフィールドは、以下のフィールドを*1つ以上*持つ別のオブジェクトを保持しています。

| フィールド      | 型   | [内部の型][] | 説明                        |
|:-----------|:-------|:------------------|:-----------------------------------|
| MemoData   | 文字列 | Blob              | 通例、メモの内容を保持する任意の16進値。 |
| MemoFormat | 文字列 | Blob              | URLで使用できる文字を表現する16進値。通例、メモのエンコード方法に関する情報を保持しています（[MIMEタイプ](http://www.iana.org/assignments/media-types/media-types.xhtml)など）。 |
| MemoType   | 文字列 | Blob              | URLで使用できる文字を表現する16進値。通例、このメモのフォーマットを定義する一意の関係（[RFC 5988](http://tools.ietf.org/html/rfc5988#section-4)に準拠）。 |

MemoTypeフィールドとMemoFormatフィールドには、以下の文字のみを使用できます。 `ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-._~:/?#[]@!$&'()*+,;=%`

`Memos`フィールドのサイズの上限は1KBです（バイナリーフォーマットでシリアル化されている場合）。

以下に、Memosフィールドが定義されているトランザクションの例を示します。

```
{
    "TransactionType": "Payment",
    "Account": "rMmTCjGFRWPz8S2zAUUoNVSQHxtRQD4eCx",
    "Destination": "r3kmLJN5D28dHuH8vZNUZpMC43pEHpaocV",
    "Memos": [
        {
            "Memo": {
                "MemoType": "687474703a2f2f6578616d706c652e636f6d2f6d656d6f2f67656e65726963",
                "MemoData": "72656e74"
            }
        }
    ],
    "Amount": "1"
}
```
## NetworkIDフィールド <a id="networkid-field"></a>
{% badge href="https://github.com/XRPLF/rippled/releases/tag/1.11.0" %}新規: rippled 1.11.0{% /badge %}

`NetworkID`フィールドは「クロスチェーン」トランザクションのリプレイ攻撃に対する保護であり、同じトランザクションがコピーされ、意図していない[ネットワーク](../../../concepts/networks-and-servers/parallel-networks.md)で実行されることを防ぎます。既存のチェーンとの互換性のため、ネットワークIDが1024以下のネットワークでは`NetworkID`フィールドを省略する必要がありますが、ネットワークIDが1025以上のネットワークでは`NetworkID`フィールドを含める必要があります。以下の表は、さまざまな既知のネットワークのステータスと値を示しています。

| ネットワーク    | ID | `NetworkID`フィールド |
|---------------|----|---------------------|
| Mainnet       | 0  | 使用不可             |
| Testnet       | 1  | 使用不可             |
| Devnet        | 2  | 使用不可             |
| AMM Devnet    | 25 | 使用不可             |
| Sidechains Devnet Locking Chain | 2551 | 使用不可, ただし、アップデート後に必要となる予定です。 |
| Sidechains Devnet Issuing Chain | 2552 | 使用不可, ただし、アップデート後に必要となる予定です。 |
| Hooks V3 Testnet | 21338 | 必須    |

トランザクションのリプレイ攻撃は理論的には可能ですが、2つ目のネットワークに特定の条件が必要です。次のすべてが真でなければなりません。

- トランザクションの送信者が2つ目のネットワーク上の資金提供アカウントである。
- 2つ目のネットワーク上の送信者の`Sequence`がトランザクションの`Sequence`と一致するか、トランザクションが第二のネットワークで利用可能な[Ticket](../../../concepts/accounts/tickets.md)を使用している。
- トランザクションが`LastLedgerSequence`フィールドを持っていないか、2つ目レジャーの現在のレジャーインデックスよりも高い値を指定している。
    - メインネットは一般的に、テストネットワークやサイドチェーンよりもレジャーインデックスが高いため、トランザクションが`LastLedgerSequence`を本来の意図通りに使用している場合、メインネットのトランザクションをサイドチェーンやテストネットワークでリプレイする方が現実的です。
- ネットワークが両方とも1024以下のIDを持っているか、両方のネットワークが同じIDを使用しているか、2つ目のネットワークが`NetworkID`フィールドを必要としないかのいずれか。


## Signersフィールド

`Signers`フィールドには、最大8つのキーペアから取得された署名を保持し、トランザクションを承認するための[マルチシグ](../../../concepts/accounts/multi-signing.md)が含まれています。`Signers`リストはオブジェクトの配列であり、各オブジェクトが1つの`Signer`フィールドを保持しています。`Signer`フィールドには、以下の入れ子フィールドがあります。

| フィールド         | 型   | [内部の型][] | 説明                     |
|:--------------|:-------|:------------------|:--------------------------------|
| Account       | 文字列 | AccountID         | SignerListに記述され、この署名に関連付けられているアドレス。 |
| TxnSignature  | 文字列 | Blob              | `SigningPubKey`を使用して検証できる、このトランザクションの署名。 |
| SigningPubKey | 文字列 | Blob              | この署名の作成に使用される公開鍵。 |

`SigningPubKey`は、`Account`アドレスに関連付けられているキーでなければなりません。参照されている`Account`が、レジャーにあり資金供給済みアカウントである場合、SigningPubKeyには、そのアカウントの現在のレギュラーキー（設定されている場合）を指定できます。また、[lsfDisableMaster](../ledger-data/ledger-entry-types/accountroot.md#accountrootのフラグ)フラグが有効になっている場合を除き、そのアカウントのマスターキーを指定することもできます。参照されている`Account`アドレスが、レジャーの資金供給済みのアカウントではない場合、`SigningPubKey`は、そのアドレスに関連付けられているマスターキーでなければなりません。

署名の検証は大量の演算能力を消費するタスクであるため、マルチシグトランザクションをネットワークに中継するには、追加のXRPがコストとしてかかります。マルチシグに含まれている署名ごとに、トランザクションに必要な[トランザクションコスト][]が増加します。例えば、トランザクションをネットワークに中継するための現在の最小トランザクションコストが`10000`dropである場合、`Signers`配列に3つのエントリが含まれているマルチシグトランザクションを中継するには、`Fee`の値を少なくとも`40000`dropにする必要があります。

{% raw-partial file="/docs/_snippets/common-links.md" /%}