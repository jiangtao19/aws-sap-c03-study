# SAP-C03 第2週 週末ハンズオン・ランブック（土 11/21・日 11/22）

作成日：2026-11-20（金）／対象：第2週（11/16〜22）の週末ハンズオン
所要：週末合計 3〜4時間（土・日 各1.5〜2時間目安）。平日シート Day 1〜Day 4 の検証編で、新しい解説は含まない。
前提：SAA 取得済み・実務経験少なめの方向けに、手順と「確認すべき結果」を先に書いてある。詰まった点と所要時間はメモしておく（後で効く記録になる）。

## 始める前の確認（両日共通・5分）

- [ ] 予算アラートが有効か確認する（`week1-hands-on-budget-setup.md`：月額コスト予算、実績 50%／80%／100%＋予測 100%、通知メールの承認済みか）。Budgets は通知のみで課金を止めない。
- [ ] 使うのはテスト用のアカウント／OU。ふだん使いのアカウントと混ぜない。root ユーザーでの日常作業はしない。
- [ ] リージョンは東京（ap-northeast-1）で統一する。

## 土曜 11/21：Organizations／OU／SCP の検証

目的：テスト用 OU に SCP を付け、「暗号化なし S3 作成・書き込みが拒否される」ことを実際に確認する。

### 手順

1. Organizations を有効化する（まだの場合のみ。すでに組織がある場合はこの手順は不要）。
2. テスト用 OU を作る（例：`TestOU`）。検証用のアカウントをこの OU に入れる。
3. SCP を1つ作る。Day 1 の CreateBucket 型と Day 2 の PutObject 型を**1つの SCP にまとめる**（暗号化強制は作成＋書き込みの2段構え。片方だけでは穴が残る）。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedBucketCreation",
      "Effect": "Deny",
      "Action": "s3:CreateBucket",
      "Resource": "arn:aws:s3:::*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["AES256", "aws:kms"]
        },
        "Null": { "s3:x-amz-server-side-encryption": "true" }
      }
    },
    {
      "Sid": "DenyUnencryptedPutObject",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::*/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["AES256", "aws:kms"]
        },
        "Null": { "s3:x-amz-server-side-encryption": "true" }
      }
    }
  ]
}
```

4. 作った SCP をテスト用 OU にアタッチする。`FullAWSAccess` は残したままにする（外すと何もできなくなる）。
5. コンソールでポリシーの構文確認をしてからアタッチする。

### 確認すること（期待する結果）

- [ ] 暗号化なしで `CreateBucket` → **拒否される**（AccessDenied）。
- [ ] 暗号化あり（AES256 または aws:kms）で `CreateBucket` → **作れる**。
- [ ] 既存バケットへ暗号化なしで `PutObject` → **拒否される**。
- [ ] 余力があれば：リージョン制限 SCP（Day 2 §1 の型）で、東京以外での EC2 起動が拒否され、IAM ユーザー作成は通ることも確認する（IAM はグローバルサービスで `NotAction` に除外されているため）。

### 一番の落とし穴

- **管理アカウントでは SCP は効かない。**検証は必ずテスト用 OU のメンバーアカウントで行う。管理アカウントで試して「拒否されない」のは、設定ミスではなく仕様。
- メンバーアカウントの root も SCP の制限を受ける。管理アカウントの root は受けない。
- 拒否されなかったら見る順番：① SCP のアタッチ先はテスト用 OU／アカウントか（管理アカウントではないか）② ポリシーの構文・条件キー（`s3:x-amz-server-side-encryption`）③ SCP の反映を数分待つ ④ 成功ケース側は IAM ポリシーでも許可されているか（SCP は許可を与えない）。
- うまくいかない場合の代替：IAM ポリシー＋AWS Config ルールでの検証に切り替える（Day 2 §4 の予防・検出・記録の整理を参照）。

### 土曜の片付け

- [ ] 検証用バケットを削除する（作ったものがあれば）。
- [ ] SCP とテスト用 OU は、日曜の作業と干渉しないため残してよい。来週以降不要になったらデタッチしてから削除する。

## 日曜 11/22：VPC 2つ＋VPC Lattice の疎通確認

目的：VPC を2つ作り、VPC Lattice でサービスを公開して DNS 名で疎通確認する。認証を `NONE` から `AWS_IAM` に切り替えて、拒否されることも確認する。

### 手順

1. VPC を2つ作る。CIDR は重なっていてもよい（例：両方 `10.0.0.0/16`）。Lattice は L7・サービス単位で、CIDR の重なりが問題にならない点の確認にもなる。ピアリングや Transit Gateway なら重なりは不可。
2. サービスネットワークを作り、2つの VPC を関連付ける。1つの VPC が直接関連付けられるサービスネットワークは1つだけ、という点も覚えておく。
3. サービスを作る（リスナー：ポート・プロトコル、ルール、ターゲットグループ）。ターゲットグループの裏側は ALB／EC2／Lambda／IP のいずれか。
4. クライアント側の VPC から、サービスの DNS 名（`xxx.vpc-lattice-svcs.…`）を呼ぶ。クライアント側に相手 VPC へのルートや VPC ピアリングは要らない。

### 確認すること（期待する結果）

- [ ] 認証 `NONE` で、DNS 名での呼び出しが**通る**。
- [ ] 認証を `AWS_IAM` に切り替える。署名なしの呼び出しが**拒否される**。
- [ ] 署名あり（SigV4、署名名 `vpc-lattice-svcs`）の呼び出しが**通る**。呼び手の IAM ポリシー（`vpc-lattice-svcs:Invoke` の許可）も要る。
- [ ] 余力があれば：Lattice のアクセスログ（CloudWatch Logs／S3）で、誰がどのサービスを呼んだかを確認する。
- [ ] 所要時間と詰まった点をメモする。

### 覚えておく認証の3つの門（どれか1つでも拒否なら通らない）

1. 呼び手の IAM ポリシー（`vpc-lattice-svcs:Invoke`）
2. サービスネットワークの認証ポリシー（粗いガードレール）
3. サービスの認証ポリシー（細かい制御。例：`checkout` ロールだけが `POST /payments` を呼べる）

### 日曜の片付け（忘れずに）

- [ ] サービスを削除する。
- [ ] サービスネットワークを削除する。
- [ ] VPC 2つを削除する。
- [ ] Cost Explorer で当月の実績を1回確認する。
- [ ] 予算アラートのチェックリストで確認する（`week1-hands-on-budget-setup.md`）。

## 週末の自己採点（答えは作らない・キックオフのチェックポイント）

1. SCP で「特定リージョンでのリソース作成を禁止」したい。どのポリシー要素でどう書くか、IAM ポリシーとの違いは何か。
2. 3つの VPC 間でサービスを公開・発見したい。VPC Lattice と Transit Gateway＋PrivateLink の使い分け条件を2つ挙げよ。
3. 開発アカウントで誤って公開 S3 バケットが作られる事故を防ぐ。予防（SCP）・検出（Config）・記録（CloudTrail）の役割分担を説明せよ。

## 関連

- 週次プラン：`week2-kickoff.md`（週末ハンズオンの概要）
- Day 1 シート：`week2-day1-organizations-scp.md`（SCP 評価ロジック、CreateBucket 型）
- Day 2 シート：`week2-day2-scp-examples.md`（リージョン制限、PutObject 型、root への影響）
- Day 3 シート：`week2-day3-vpc-lattice.md`（Lattice の骨格、認証の3つの門、使い分け表）
- Day 4 シート：`week2-day4-hybrid-dns.md`（ハイブリッド接続と DNS。L3 と L7 の対比）
- 予算アラート：`week1-hands-on-budget-setup.md`（ハンズオン前のガードレール、片付けチェックリスト）
- 合格ライン：720/1000（予約時に公式ガイドで再確認）。受験料 300 USD。
