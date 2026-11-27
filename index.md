# AWS SAP-C03 学習ノート

SAA取得済み・実務経験少なめの前提で、SAP-C03（2026-11-17〜）合格を目指す学習記録です。受験目標：2027年2月中旬（2/8〜14の週）。平日1時間＋週末3〜4時間のペースで進めています。

- 試験形式：75問・180分・受験料 300 USD・日本語可・合格 720/1,000（SAP-C03公式ガイドの記載。予約時に最終確認）
- 分野配分：①アーキテクチャ設計と実装 25%／②セキュリティ・コンプライアンス・ガバナンス 25%／③コスト最適化 15%／④回復力・移行・事業継続性 15%／⑤運用上の優秀性とオートメーション 20%
- 注意：これは個人の学習ノートで、公式問題ではありません。認証情報などは含まれていません。

## 目次

### 第1週（11/9〜15）：現在地診断と準備

| シート | 内容 |
| --- | --- |
| [week1-diagnostic.md](week1-diagnostic.md) | 現在地診断ミニ問題（10問）— 5分野の分野別スコアで弱点を確認 |
| [week1-hands-on-budget-setup.md](week1-hands-on-budget-setup.md) | ハンズオン用アカウントの予算ガードレール（約10分）— Budgetsで月額予算と50%／80%／100%＋予測100%の通知 |

### 第2週（11/16〜22）：Organizations／SCPとネットワーク

| シート | 内容 |
| --- | --- |
| [week2-kickoff.md](week2-kickoff.md) | 第2週キックオフ — 今週のテーマと平日・週末の計画 |
| [week2-day1-organizations-scp.md](week2-day1-organizations-scp.md) | Day 1（11/16）：OrganizationsとSCP — OU設計、SCPの評価ロジック |
| [week2-day2-scp-examples.md](week2-day2-scp-examples.md) | Day 2（11/17）：SCPの実例 — リージョン制限、暗号化強制、ルートユーザーへの影響 |
| [week2-day3-vpc-lattice.md](week2-day3-vpc-lattice.md) | Day 3（11/19）：VPC Lattice — サービスネットワーク、IAM認証の二層構造、PrivateLink等との比較 |
| [week2-day4-hybrid-dns.md](week2-day4-hybrid-dns.md) | Day 4（11/20）：ハイブリッド接続とDNS — Direct Connect／Site-to-Site VPN、Route 53 Resolver |
| [week2-weekend-hands-on-runbook.md](week2-weekend-hands-on-runbook.md) | 週末ハンズオン・ランブック（11/21 SCP・11/22 VPC Lattice）— 今日・明日の検証手順と片付けチェックリスト |

### 第3週（11/23〜29）：コンピュート／サーバーレスとデータ設計

| シート | 内容 |
| --- | --- |
| [week3-kickoff.md](week3-kickoff.md) | 第3週キックオフ — Lambda、ECS／EKS、Step Functions、Aurora／RDS、DynamoDB／S3、週末はサーバーレスパイプラインのハンズオン |
| [week3-day1-lambda-serverless.md](week3-day1-lambda-serverless.md) | Day 1（11/23）：Lambda／サーバーレスの基礎 — 15分制限、同期・非同期・ストリームのリトライと冪等性 |
| [week3-day2-ecs-eks-fargate.md](week3-day2-ecs-eks-fargate.md) | Day 2（11/24）：ECS／EKSとFargate — タスク実行ロールとタスクロールの違い、起動タイプの使い分け |
| [week3-day3-stepfunctions-eventbridge-sqs-sns.md](week3-day3-stepfunctions-eventbridge-sqs-sns.md) | Day 3（11/25）：Step Functions／EventBridge／SQS／SNS — StandardとExpress、Retry／Catch、使い分け表 |
| [week3-day4-aurora-rds.md](week3-day4-aurora-rds.md) | Day 4（11/26）：Aurora／RDS — Multi-AZとリードレプリカ、Serverless v2、Global Database |
| [week3-day5-dynamodb-s3.md](week3-day5-dynamodb-s3.md) | Day 5（11/27）：DynamoDBとS3の設計 — パーティションキー設計、ストレージクラスとライフサイクル、週末ハンズオンの準備メモ |
| [week3-weekend-hands-on-runbook.md](week3-weekend-hands-on-runbook.md) | 週末ハンズオン・ランブック（11/28 S3→Lambda→DynamoDBの部品確認・11/29 Step Functionsでつないで失敗も確認）— 検証手順と片付けチェックリスト |

## 進め方

- 各Dayシートは60分フォーマット（読む＋問題＋振り返り）です。
- ハンズオンの前には予算アラートの確認、後にはリソース削除とCost Explorer確認をします。
- 診断の分野別スコアが分かり次第、弱い分野を前に倒して計画を組み替えます。
