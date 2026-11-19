# SAP-C03 第2週 Day 3（木 11/19）学習シート：VPC Lattice

所要：60分（復習10分＋概念・認証25分＋使い分け10分＋問題15分）。分野①（アーキテクチャ設計と実装）／分野②（セキュリティ/コンプライアンス/ガバナンス）。C03 の新規範囲として重点。日曜の週末ハンズオン（VPC 2つ＋VPC Lattice の疎通確認）の予習にもなる。

## 0. 復習：Day 1・Day 2 と水曜のネットワーク基礎（10分）

- SCP は「許可」ではなく「上限（ガードレール）」。IAM で許可されていても、SCP で拒否されれば使えない。管理アカウントには効かない。
- 暗号化強制は「作成（CreateBucket）＋書き込み（PutObject）」の2段構え。片方だけでは穴が残る。
- 水曜の基礎：VPC 間の接続はふつう、ルートテーブルと CIDR が要る。セキュリティグループはインスタンス単位、NACL はサブネット単位。今日の Lattice は、この「ルートと CIDR が要る」という前提をひっくり返す。

## 1. Lattice の骨格：サービスネットワークが「境界」になる（15分）

VPC Lattice は、VPC やアカウントをまたいでサービス同士をつなぐ、マネージドな**アプリケーション・ネットワーキング（L7、HTTP/HTTPS/gRPC）**の仕組み。つなぐ単位はサブネットや CIDR ではなく**サービス**。

- **サービスネットワーク**：サービスと VPC を集める論理的な境界。CIDR の意味での「ネットワーク」ではない。ここにサービスを関連付け、クライアント側の VPC を関連付ける。
- **サービス**：公開したい単位。リスナー（ポート・プロトコル）とルール（パスやヘッダーによる振り分け）、ターゲットグループを持つ。
- **ターゲットグループ**：サービスの裏側。ALB、EC2 インスタンス、Lambda、IP アドレスなどを登録する。
- **呼び方**：関連付けられた VPC の中から、サービスの DNS 名（`xxx.vpc-lattice-svcs.…`）を呼ぶだけ。クライアント側に、相手 VPC へのルートや VPC ピアリングは要らない。Lattice がルーティングと負荷分散をする。

押さえるポイントは3つ。

1. **CIDR が重なっていてもつながる**。ピアリングや Transit Gateway は L3 でルートを張るため、CIDR の重なりが許されない。Lattice は L7 でサービス名宛てに振り分けるため、重なりが問題にならない。試験では「CIDR が重なった2つの VPC 間でサービスを公開したい」が Lattice の合図。
2. **多対多が1つの境界で済む**。PrivateLink は「1サービスにつき1エンドポイント」が基本で、サービスが増えるとエンドポイントも増える。Lattice は1つのサービスネットワークに複数のサービスと複数の VPC を入れて、多対多の呼び合いができる。
3. **サービスネットワークは RAM で共有できる**。AWS Resource Access Manager（RAM）で、同じ Organizations 内の別アカウントとサービスネットワークやサービスを共有する。Day 1 の OU 設計とつながる話で、組織をまたぐサービス公開の土台になる。

## 2. 認証は2層＋呼び手の IAM、3つの門を全部通る（10分）

Lattice の認証タイプは `NONE` か `AWS_IAM`。`AWS_IAM` の場合、呼び手は SigV4 で署名する（署名名 `vpc-lattice-svcs`）。評価はこの順で、**どれか1つでも拒否なら通らない**。ポリシーが1つも無い場合は暗黙の拒否。

1. 呼び手の IAM ポリシー：`vpc-lattice-svcs:Invoke` を許可していること。
2. サービスネットワークの認証ポリシー：ネットワーク全体の粗いガードレール。例：「この Organizations のプリンシパルだけ」。
3. サービスの認証ポリシー：サービス単位の細かい制御。例：「`checkout` ロールだけが `POST /payments` を呼べる」。

サービス認証ポリシーの型（アカウント・ロールは自分のものに置き換えること）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/checkout-svc" },
      "Action": "vpc-lattice-svcs:Invoke",
      "Resource": "arn:aws:vpc-lattice:ap-northeast-1:111122223333:service/svc-0123456789abcdef0",
      "Condition": {
        "StringEquals": { "vpc-lattice-svcs:RequestMethod": "POST" }
      }
    }
  ]
}
```

- 条件キーには `vpc-lattice-svcs:RequestMethod`、`vpc-lattice-svcs:RequestPath`、`vpc-lattice-svcs:SourceVpc` などがあり、メソッドやパス、呼び元 VPC で絞れる。
- セキュリティグループや NACL が「IP・ポートの出入り」を制御するのに対し、Lattice の認証ポリシーは「**どの IAM プリンシパルが、どのサービスを、どう呼べるか**」を制御する。両者は置き換えではなく重ねるもの。
- 監視は CloudWatch メトリクスとアクセスログ（S3／CloudWatch Logs／Firehose へ）で、誰がどのサービスを呼んだかを追える。週末ハンズオンでは、疎通確認のあとログが残ることも見ておく。

## 3. 使い分け：Lattice／PrivateLink／Transit Gateway／ピアリング（10分）

| 観点 | VPC Lattice | PrivateLink | Transit Gateway | VPC ピアリング |
|---|---|---|---|---|
| 層・単位 | L7、サービス単位 | L3/L4、エンドポイント単位 | L3、VPC 単位（ハブ＆スポーク） | L3、VPC 単位（1対1） |
| つなぎ方 | サービスネットワークに関連付け | サービスごとに VPC エンドポイント | ハブに VPC・オンプレを接続 | VPC 同士を直接接続 |
| CIDR の重なり | 問題にならない | エンドポイント経由のため重なりを避けやすい | 重なり不可 | 重なり不可 |
| 認証 | IAM 認証ポリシー（SigV4） | エンドポイントポリシー＋SG | ルート＋SG/NACL（IAM の認証はない） | ルート＋SG/NACL |
| 向く場面 | 複数 VPC・アカウント間のサービス呼び合い、マイクロサービス | 特定サービスを1つの VPC に非公開で出す | 多数 VPC＋オンプレの一元接続 | 2つの VPC の単純接続 |

試験の見分け方：**「CIDR が重なっている」「サービス単位で、誰が呼べるかまで制御したい」「多対多」**なら Lattice。**「多数の VPC とオンプレを L3 でつなぐ」**なら Transit Gateway。**「1つのサービスだけを、相手 VPC から IP を出さずに使わせたい」**なら PrivateLink。

## 4. ミニ問題（15分、答えは下）

**Q1**：VPC-A と VPC-B の CIDR がどちらも `10.0.0.0/16` で重なっている。VPC-B にある注文サービスを VPC-A から呼びたい。VPC ピアリングと Transit Gateway ではなぜ難しいか。Lattice ならどう解決するか。

**Q2**：サービスネットワークの認証ポリシーは「同じ Organizations のプリンシパルなら可」、サービスの認証ポリシーは「`checkout` ロールのみ可」とした。同じ Organizations 内の `orders` ロールがこのサービスを呼んだらどうなるか。呼び手の IAM ポリシーも含め、3つの門で説明せよ。

**Q3**：決済サービスで「`POST /payments` は `checkout` ロールだけ、それ以外は不可」としたい。セキュリティグループだけで実現できるか。Lattice なら何を使い、呼び出しの記録はどこで確認するか。

### 答え・解説

- **A1**：ピアリングと Transit Gateway は L3 でルートを張るため、宛先 CIDR が重なるとどちらの VPC か判別できず、接続できない。Lattice は L7 でサービス名宛てに振り分け、クライアント側に相手 VPC へのルートが要らないため、CIDR の重なりが問題にならない。サービスネットワークに両 VPC を関連付け、DNS 名で呼ぶ。
- **A2**：拒否される。3つの門はすべて通る必要がある。呼び手の IAM ポリシーで `vpc-lattice-svcs:Invoke` が許可されていても、サービス認証ポリシーは `checkout` ロールのみ許可で、`orders` ロールは通らない。ネットワーク側の「Organizations 内なら可」は粗い門で、サービス側の細かい門を緩められない。明示的拒否があればなおさら通らない。
- **A3**：セキュリティグループだけではできない。SG は IP・ポート単位で、IAM プリンシパルや HTTP メソッド・パスは見ない。Lattice のサービス認証ポリシーで、Principal を `checkout` ロール、Action を `vpc-lattice-svcs:Invoke`、条件に `vpc-lattice-svcs:RequestMethod = POST` と `vpc-lattice-svcs:RequestPath = /payments` を組み合わせる。呼び出しの記録は Lattice のアクセスログ（CloudWatch Logs／S3）で確認する。

## 5. 振り返りメモ（残り時間）

- 今日の3行サマリーを自分の言葉で書く（例：「Lattice は L7・サービス単位／CIDR の重なりは問題にならない／認証は呼び手IAM＋ネットワーク＋サービスの3つの門」）。
- 日曜のハンズオンの検証項目を決める：「VPC を2つ作り、サービスネットワークに関連付けて DNS 名で疎通確認。認証はまず `NONE` で疎通を確認し、余力があれば `AWS_IAM` に切り替えて、署名なしの呼び出しが拒否されることも確認」。
- 片付け：使い終わったらサービス、サービスネットワーク、VPC 2つを削除し、予算アラートのシートのチェックリストで確認する。

## 関連

- 週次プラン：`week2-kickoff.md`（木 11/19 は VPC Lattice、日曜は VPC 2つ＋Lattice のハンズオン。チェックポイント Q2 は Lattice と Transit Gateway＋PrivateLink の使い分け）
- Day 1 シート：`week2-day1-organizations-scp.md`（OU・SCP の評価ロジック）
- Day 2 シート：`week2-day2-scp-examples.md`（リージョン制限・暗号化強制の型）
- 予算アラート：`week1-hands-on-budget-setup.md`（ハンズオン前のガードレール、片付けチェックリスト）
- 合格ライン：720/1000（予約時に公式ガイドで再確認）。受験料 300 USD、SAA 50% バウチャー期限は未報告。
