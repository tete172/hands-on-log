# 2026-09-19 VPC / EC2 / RDS / Secrets Manager を手で構築する

## ゴール

ダッシュボード(FastAPI)を載せるネットワークとサーバを、**設計から自分で作る**。ボット用の既存 EC2・VPC には触らない。RDS は「構築して接続確認する」ところまでを実績にし、継続運用は SQLite にしてコストを抑える。

## 全体構成

```
dashboard-vpc (10.1.0.0/16)
├── public subnet  ×2 (1a / 1c)   ← EC2 を置く。IGW への経路あり
├── private subnet ×2 (1a / 1c)   ← RDS を置く。インターネットへの経路なし
├── Internet Gateway
└── (NAT Gateway は作らない = RDS は外向き通信が不要でコストがかかるため)

EC2 (t3.micro, public subnet) ──IAM ロール──▶ Secrets Manager(DB 認証情報)
        │
        └── 5432 ──▶ RDS PostgreSQL (private subnet, 非公開)
```

## 実施内容

### 1. ドメイン取得
- Route 53 で独自ドメインを取得($16/年、`.com`)。ホストゾーンは登録と同時に自動作成される
- 取得できるドメインが `.com` で $16 だったのは `list-prices` で事前確認した

### 2. VPC
- `10.1.0.0/16` を新規作成。既存のボット用 VPC(`172.31.0.0/16`)・学習用 VPC(`10.0.0.0/24`)と CIDR が重ならないようにした
- 「VPC など」ウィザードで作成。結果は次のとおり
  - public ×2: `10.1.0.0/20`(1a)、`10.1.16.0/20`(1c)
  - private ×2: `10.1.128.0/20`(1a)、`10.1.144.0/20`(1c)
  - public 用ルートテーブル: `0.0.0.0/0 → IGW`
  - private 用ルートテーブル: local のみ(+ ウィザードが作った S3 ゲートウェイエンドポイント。無料)
- **VPC を分けた理由**: 動いている本番ボットのネットワークを、作業ミスで壊すリスクをなくすため。VPC 自体は無料

**詰まった点**
- ウィザードで作った public サブネットの「パブリック IPv4 アドレスの自動割り当て」が **OFF** だった。サブネットごとに個別に ON にする必要がある(複数選択すると「サブネットの設定を編集」がグレーアウトする)。private は OFF のままが正しい
- サブネットが一覧に出ないと思ったら、コンソールの表示が古いだけだった(更新アイコンで解決)

### 3. セキュリティグループ
| SG | インバウンド | 意図 |
|---|---|---|
| EC2 用 | 22 / 80 / 443 を **自宅 IP の `/32` のみ** | 世界に公開しない |
| RDS 用 | 5432 を **EC2 用 SG からのみ**(IP 指定ではなく SG 参照) | EC2 以外からは到達できない |

**学び**: SG の「説明」欄は日本語不可(英数字と一部記号のみ)。日本語で分かりやすくしたい場合は Name タグを使う。

### 4. IAM ロール(最小権限)
IAM ロールは 2 層でできている。

- **信頼ポリシー**: 誰がこのロールを使えるか → `ec2.amazonaws.com` のみ
- **許可ポリシー**: そのロールで何ができるか → 特定 1 つのシークレットに対する `secretsmanager:GetSecretValue` のみ

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue"],
    "Resource": ["arn:aws:secretsmanager:ap-northeast-1:<ACCOUNT_ID>:secret:dashboard/db-credentials-xxxxxx"]
  }]
}
```

**詰まった点**: 最初ロール作成画面のポリシーで `Resource` に入れる ARN が決まらず止まった。**先に Secrets Manager でシークレットを作って ARN を得てから、ロールにインラインポリシーを付ける**順番が正しい(ロールだけ先に作っておき、後からポリシーを付けるのでも可)。

### 5. Secrets Manager
- シークレット名 `dashboard/db-credentials`、キーは `DB_USERNAME` / `DB_PASSWORD`。パスワードはランダム生成(24 文字)
- 自動ローテーションは **OFF**。ローテーションには専用の Lambda が必要でミニマム構成には過剰なため、今回は見送り(今後の課題)
- レビュー画面で値が表示されないのは仕様(セキュリティ上デフォルトでマスクされる)

**設計判断: SSM Parameter Store ではなく Secrets Manager にした**
| | SSM Parameter Store (SecureString) | Secrets Manager |
|---|---|---|
| 費用 | 無料(標準パラメータ) | 1 シークレット $0.40/月 |
| 自動ローテーション | なし | あり(Lambda) |
| 採用理由 | | 月 $0.40 は許容範囲で、シークレット管理の標準サービスとして経験しておきたい |

### 6. EC2
- Amazon Linux 2023 / t3.micro / public subnet / EC2 用 SG / IAM インスタンスプロファイルをアタッチ

**詰まった点(重要)**
- 1 回目、サブネットの選択を間違えて **private subnet に起動**してしまった。パブリック IP は付いたが、private のルートテーブルに IGW への経路が無いので **SSH もインターネットも通らない**
- インスタンスのサブネットは起動後に変更できない → **終了して public subnet で作り直した**
- 教訓: 起動前に「サブネット名(public / private)」と CIDR を見て選ぶ。起動後は `describe-instances` でサブネット ID を確認する

**動作確認**
- SSH 接続できる(SG の 22 が自宅 IP のみで効いている)
- EC2 上で認証情報を一切設定せずに次のコマンドが通った → **IAM ロールが効いている**

```bash
aws secretsmanager get-secret-value --secret-id dashboard/db-credentials --region ap-northeast-1 --query SecretString --output text
```

### 7. RDS(検証用・一時利用)
| 項目 | 設定 | 理由 |
|---|---|---|
| エンジン | PostgreSQL | アプリの想定 DB |
| テンプレート | 開発/テスト | |
| 可用性 | シングル AZ | 検証用にマルチ AZ は不要(コスト 2 倍) |
| インスタンスクラス | **db.t3.micro**(バースト可能クラス) | 初期表示は `db.m7g.large` で高額だった |
| ストレージ | gp3 **20 GiB** | 初期値は 200 GiB だった |
| サブネットグループ | 新規作成(private ×2) | RDS は 2AZ 以上必須 |
| パブリックアクセス | なし | |
| SG | RDS 用 SG | |
| 認証情報 | **セルフマネージド**(自分のシークレットと同じ値を入力) | 「Secrets Manager で管理」を選ぶと RDS が**別のシークレットを自動作成**し、追加料金が発生し、既存のシークレット/IAM ポリシーとも噛み合わなくなる |
| 最初の DB 名 | `fxbot_dashboard` | 空欄だとインスタンスだけ立ってデータベースは作られない |
| バックアップ / 削除保護 | OFF | 検証後に確実に消せるように |

**コストの見方**: 作成画面の「概算月間コスト」($23.20)は **1 か月動かし続けた場合**の見積り。課金は時間単位なので、検証して削除すれば実費は数十円〜。逆に消し忘れると月 $23 が発生し続ける。なおアカウント開設から 3 年経っており、12 か月の無料枠は対象外。

**動作確認**
```bash
sudo dnf install -y postgresql16          # psql クライアント
psql -h <RDS エンドポイント> -U dashboard_admin -d fxbot_dashboard
\dt   # → Did not find any relations.(新品の DB なので正常)
```
EC2(public)→ RDS(private)の接続と、Secrets Manager に入れた認証情報が RDS の実際の認証情報と一致していることを確認できた。

**削除**: 確認後すぐ削除(最終スナップショットなし)。`describe-db-instances` が `DBInstanceNotFound`、スナップショット 0 件で課金対象が残っていないことを確認。

> RDS の用語: 「DB インスタンス」= RDS のデータベースサーバ 1 台のこと。削除しても EC2 / VPC / SG / IAM / シークレットには影響しない。

## 今日の学び(まとめ)

1. **public / private の違いはルートテーブル**で決まる(サブネットの名前ではない)。IGW への経路があるかどうか
2. **SG は「SG 参照」でつなぐ**とアドレスに依存しない最小権限にできる
3. **IAM ロールは「誰が」(信頼ポリシー)と「何を」(許可ポリシー)の 2 層**。キーを置かずに AWS リソースへ到達できる
4. **依存関係のある作成順序**がある(シークレット → ARN → IAM ポリシー)
5. **コスト見積りは「フル稼働の月額」**で、検証用は使ったら消す

## 次

- Elastic IP を付け、Route 53 の A レコードでドメインを向ける
- nginx(リバースプロキシ)+ Let's Encrypt で HTTPS 化。標準の ACM 公開証明書は ALB / CloudFront など AWS 統合サービス向けで EC2 に直接置けない(エクスポート可能な証明書は有料オプション)ため、単一 EC2 では Let's Encrypt を使う
- アプリ(uvicorn)を systemd で常駐化
