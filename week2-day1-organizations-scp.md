# SAP-C03 第2週 Day 1（月 11/16）学習シート：AWS Organizations と SCP

所要：60分（読む20分＋問題20分＋振り返り20分）。分野②（セキュリティ/コンプライアンス/ガバナンス）／週末ハンズオンの土台。

## 1. Organizations の骨格（10分で押さえる）

- **組織（organization）**：管理アカウント（旧マスター）を1つ持ち、メンバーアカウントを束ねる。SCP は「管理アカウントのルートポリシー → ルート → OU（入れ子可）→ アカウント」というツリーで評価される。
- **OU**：アカウントのグループ。SCP は OU にアタッチでき、子 OU とその全アカウントに継承される。設計の原則は「ガバナンス単位＝OU」。例：`Prod / Dev / Sandbox / Security` のように、統制の強さを変えたい単位で切る。
- **アクセス**：管理アカウントは AWS Single Sign-On（IAM Identity Center）で一元管理するのが C03 の定石。メンバーアカウントへの直接の root 利用は避ける。

## 2. SCP の評価ロジック（ここが試験の山）

1. **SCP は「許可」ではなく「上限（ガードレール）」**。SCP が許可しても、IAM ポリシーで許可されなければ使えない。逆に SCP で拒否されれば、IAM が許可していても使えない。
2. **明示的拒否（Explicit Deny）が最優先**。どのレベルの SCP でも Deny が1つでもマッチすれば即拒否。Allow は全レベルで揃って初めて通る（論理 AND）。
3. **既定の FullAWSAccess**：ルートには `FullAWSAccess`（全許可）が最初からアタッチされている。これを外すと何もできなくなるので、実務では「FullAWSAccess を残したまま Deny ポリシーを追加」が基本形。
4. **管理アカウントには SCP が効かない**。SCP はメンバーアカウントにのみ適用される。試験で頻出のひっかけ。
5. **サービスリンクロール（SLR）や一部の例外**：SCP は SLR の操作を制限しない。AWS のドキュメントで例外リストを確認しておく（例：特定のサービスが SCP を回避するケース）。
6. **条件キー**：リージョン制限は `aws:RequestedRegion`、タグ条件は `aws:PrincipalTag` / `aws:ResourceTag` / `aws:TagKeys`、暗号化ヘッダーは `s3:x-amz-server-side-encryption`。

## 3. IAM ポリシーとの違い（対比表）

| 観点 | SCP | IAM ポリシー |
|---|---|---|
| 対象 | アカウント（メンバー）全体 | ユーザー／ロール／グループ |
| 効果 | 制限のみ（許可できない） | 許可・拒否の両方 |
| root ユーザー | 管理アカウントの root には無効 | 適用される |
| 典型用途 | 組織横断の禁止事項（リージョン封じ、暗号化強制） | 業務ごとの権限付与 |

## 4. ポリシーの型（コンソールで検証してから使う）

暗号化なし S3 バケット作成を拒否する型（週末ハンズオンで使う）：

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
    }
  ]
}
```

読み方：暗号化ヘッダーが「無い（Null）」か「AES256/aws:kms のどちらでもない」場合に Deny。タグやリージョンの型も同じ構造（`aws:RequestedRegion` に `StringNotEquals`）で書ける。※ポリシーは Organizations コンソールで構文検証してからアタッチすること。

## 5. ミニ問題（20分、答えは下）

**Q1**：開発 OU に「ap-northeast-1 以外でのリソース作成禁止」の SCP をアタッチした。開発アカウントの管理者が IAM で AdministratorAccess を付けても、us-east-1 で EC2 を起動できない。なぜか。

**Q2**：管理アカウントの root ユーザーが SCP の制限を回避できるのはなぜか。メンバーアカウントの root ユーザーも同じか。

**Q3**：OU に SCP を2つ（ポリシーA：S3 許可、ポリシーB：特定バケットへの Deny）アタッチした。対象バケットへの PutObject はどうなるか。理由も。

### 答え・解説

- **A1**：SCP は IAM の許可を上書きする上限だから。SCP が Deny（または許可の欠落）なら、IAM の Allow は無効。リージョン制限は `aws:RequestedRegion` への StringNotEquals で実装するのが定石。
- **A2**：SCP はメンバーアカウントにのみ適用され、管理アカウント（とその root）には一切効かないため。メンバーアカウントの root は SCP の制限を受ける（ただし root の利用自体を避けるのがベストプラクティス）。
- **A3**：拒否される。複数の SCP は論理 AND で評価され、どれか1つでも Deny がマッチすれば明示的拒否が最優先で勝つ。ポリシーA の Allow は他の条件を満たす操作にのみ効く。

## 6. 振り返りメモ（残り20分）

- 今日の3行サマリーを自分の言葉で書く（例：「SCP は許可しない、拒否だけ／管理アカウントは対象外／Deny が最優先」）。
- 週末ハンズオンの検証項目を1つ決める：「テスト OU に上の暗号化 Deny を付け、暗号化なしで CreateBucket が拒否されることを確認」。

## 関連
- 週次プラン：`week2-kickoff.md`
- 合格ライン：720/1000（予約時に公式ガイドで再確認）。受験料 300 USD、SAA 50% バウチャー期限は未報告。
