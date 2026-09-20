# ハンズオン学習記録

クラウドエンジニアとして必要な **AWS / Linux / Python** を、実際に手を動かして身につけていく記録。構築した内容・詰まった点・学びを、その日のうちに残す。

当面の主軸は AWS Solutions Architect – Professional(SAP)取得。ハンズオンで触れた範囲を試験範囲と対応づけて整理していく。

## 進め方

- 構築・実行は自分で AWS コンソール / CLI / ターミナルを操作して行う
- 設計の相談・コマンドの意味の確認・レビューには AI アシスタント(Claude Code)を使う。ただし実装と記録は自分の手と言葉で行う
- 「なぜそうしたか」を説明できる状態にするため、設計判断(選択肢と理由)も一緒に書く
- 秘密情報(アカウント ID・パブリック IP・ARN の実値・認証情報)は載せない。必要な箇所は `<ACCOUNT_ID>` のようなプレースホルダにする

## ログ一覧

| 日付 | 分野 | テーマ | 内容 |
|---|---|---|---|
| 2026-09-19 | AWS | [VPC / EC2 / RDS / Secrets Manager](logs/2026-09-19_vpc-ec2-rds-secrets.md) | 新規 VPC(パブリック/プライベート×2AZ)、SG、IAM ロール(最小権限)、Secrets Manager、EC2、RDS(検証のみ・削除済み) |

分野ごとの予定:

| 分野 | これから記録する内容 |
|---|---|
| AWS | Elastic IP・Route 53・HTTPS 化、その後 SAP の範囲(可用性設計・ネットワーク・セキュリティ・コスト最適化)に沿ったハンズオン |
| Linux | systemd(service / timer)、プロセス調査(`ps` / `kill` / `journalctl`)、cron、nginx、パッケージ管理、権限まわりを自分で手を動かして |
| Python | 運用スクリプト・API 連携・テスト。ダッシュボード(FastAPI)側の実装で得た学び |

## 題材

自分で運用している FX 自動売買ボットの監視ダッシュボード([backtest-parity-dashboard](https://github.com/tete172/backtest-parity-dashboard))を AWS 上にデプロイする過程を題材にしている。ボット本体が動いている環境とは **別の VPC** に作り、本番への影響をゼロにしている。
