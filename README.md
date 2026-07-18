# econ-cal-sync

> 🇺🇸 [English README is here](README-EN.md)

高重要度の経済指標イベントを毎週自動的にGoogleカレンダーへ同期するツールです。

データソースは**プラガブル**設計で、環境変数ひとつで切り替え可能です。デフォルトのデータソース（[ForexFactory](https://www.forexfactory.com/)）は API キー不要で使えます。

> **推奨する実行方法:** 現在、GitHub ActionsからForex Factoryへアクセスすると
> HTTP 403で拒否されます。また、FMP Economic Calendarは有料プランが必要で、
> 無料APIキーではHTTP 402になります。そのため、APIキー不要のForex Factoryを
> ローカルPCから取得し、**systemd timerで毎週実行する構成を推奨します**。
> 設定方法は[ローカルでForex Factoryを使用する](#ローカルでforex-factoryを使用する)を参照してください。

---

## 概要

毎週月曜日の朝7:00 JSTにローカルのsystemd timer（または設定したスケジューラ）が起動し、設定した国・通貨（デフォルト: `USD`・`JPY`）の今後4週間分の経済指標イベントを取得してGoogleカレンダーへupsertします。`extendedProperties`を使った重複チェックにより、同じイベントを何度登録しても冪等に動作します。

### 対応データソース

| 名称                    | 環境変数 `EVENT_SOURCE`         | API キー |
|-------------------------|---------------------------------|----------|
| Forex Factory           | `forexfactory` *(デフォルト)*   | 不要     |
| Financial Modeling Prep | `fmp`                           | **有料プランが必要** (`FMP_API_KEY`) |

> **FMPについて:** FMPの無料Basic APIキーでEconomic Calendar APIを呼ぶと
> HTTP 402（Payment Required）になります。FMPを利用する場合は、Economic
> Calendarを利用できる有料プランが必要です。

> 新しいデータソースを追加するには `src/fetchers/` に小さなフェッチャークラスを実装するだけです。  
> → [新しいデータソースの追加方法](#新しいデータソースの追加方法)

---

## 技術スタック

| 区分               | 技術                                                                                          |
|--------------------|-----------------------------------------------------------------------------------------------|
| 言語               | Python 3.14+                                                                                  |
| パッケージマネージャ | [uv](https://docs.astral.sh/uv/)                                                             |
| CI / 自動化        | [GitHub Actions](https://docs.github.com/en/actions)                                         |
| カレンダー API     | [Google Calendar API v3](https://developers.google.com/calendar/api/guides/overview)          |
| 認証               | Google サービスアカウント（`google-auth` 使用）                                               |
| デフォルトデータソース | [ForexFactory](https://www.forexfactory.com/)（`market-calendar-tool` による HTML スクレイピング） |
| オプションデータソース | [Financial Modeling Prep API](https://financialmodelingprep.com/)                         |

---

## 自分の環境で使うには

### 1. リポジトリをフォークする

1. このリポジトリページ右上の **Fork** ボタンをクリックします。
2. 必要に応じてローカルにクローンします（以降の手順は GitHub の Web UI だけでも完結します）。

### 2. Google Cloud – サービスアカウントと Calendar API の設定

1. [Google Cloud Console](https://console.cloud.google.com/) を開きます。
2. 新しいプロジェクトを作成するか、既存のプロジェクトを選択します。
3. **Google Calendar API** を有効化します  
   （*APIs & Services → Library → 「Google Calendar API」で検索*）。
4. **サービスアカウント**を作成します  
   （*IAM & Admin → Service Accounts → Create Service Account*）。
5. サービスアカウントの JSON キーを生成してダウンロードします  
   （*Keys → Add Key → Create new key → JSON*）。

### 3. Google カレンダーをサービスアカウントと共有する

1. [Google カレンダー](https://calendar.google.com/) を開き、対象のカレンダーの設定画面へ移動します。
2. **設定 → 特定のユーザーと共有** を選択します。
3. サービスアカウントのメールアドレス（`@<project>.iam.gserviceaccount.com` で終わる形式）を追加し、ロールを **「予定の変更」（Editor）** に設定します。
4. *カレンダーを統合* 欄に表示される **カレンダー ID** を控えておきます  
   （例: `abc123@group.calendar.google.com` や Gmailアドレス）。

### 4. GitHub Secrets を設定する

フォーク先のリポジトリで **Settings → Secrets and variables → Actions** へ進み、以下のシークレットを追加します：

| シークレット名          | 値                                                          |
|------------------------|-------------------------------------------------------------|
| `GOOGLE_SA_JSON`       | ダウンロードしたサービスアカウント JSON ファイルの**全内容** |
| `GOOGLE_CALENDAR_ID`   | 手順 3 で控えたカレンダー ID                                |
| `FMP_API_KEY`          | FMP Economic Calendarを利用できる**有料プラン**のAPIキー（FMP使用時のみ） |

> **メモ:** デフォルトのデータソース（ForexFactory）は API キー不要です。  
> 別のデータソースに切り替える場合は、対応する API キーをシークレットに追加し、ワークフローの環境変数として渡してください。

> **⚠️ 通知設定について:** Google Calendar APIの制限により、通知タイミングはユーザー自身のカレンダー設定が適用されます。
>
> **カレンダーアプリ側で通知を 40分前・10分前に手動設定してください。**
> 設定方法: Google カレンダー → 設定（⚙️）→ 対象カレンダーを選択 → 通知 → 分数を入力

### 5. GitHub Actions を有効化する

フォーク後、GitHub Actions のワークフローがデフォルトで無効になっている場合があります。  
フォーク先の **Actions** タブを開き、**「I understand my workflows, go ahead and enable them」** をクリックして有効化してください。

> **注意:** 現在はGitHub ActionsからForex FactoryへのアクセスがHTTP 403で拒否されるため、実運用にはローカルのsystemd timerを推奨します。Actionsはテスト用途、またはEconomic Calendarを利用できる有料FMPプランを使用する場合に限って利用してください。

---

## データソースの切り替え

`.github/workflows/sync.yml` の `EVENT_SOURCE` 環境変数を変更します：

```yaml
env:
  EVENT_SOURCE: forexfactory   # 別の登録済みソース名に変更
```

リポジトリに含まれる現在のワークフローは`fmp`を指定しています。Forex Factoryをローカルで使う場合は、ローカルの`.env`だけを`forexfactory`にしてください。

---

## 手動実行

**Actions → Sync Economic Calendar → Run workflow** から、スケジュールを待たずにすぐ実行できます。

---

## ローカルでForex Factoryを使用する

Forex Factoryの将来週スクレイピングはGitHub ActionsからHTTP 403になる場合があります。
ローカルPCからは取得できるため、ローカル実行では`EVENT_SOURCE=forexfactory`を使用できます。

### 1. `.env`を作成する

サンプルをコピーします。

```bash
cp .env.example .env
```

`.env`に以下を設定します。

```dotenv
EVENT_SOURCE=forexfactory
GOOGLE_CALENDAR_ID=xxxxxxxx@group.calendar.google.com
GOOGLE_SA_JSON='{"type":"service_account", ...}'
```

`GOOGLE_SA_JSON`には、サービスアカウントJSONの全内容を1行にして設定します。
`jq`がある場合は次のコマンドで1行化できます。

```bash
jq -c . /path/to/service-account.json
```

`.env`と`service-account*.json`は`.gitignore`対象です。秘密情報をcommitしないでください。

このアプリケーションは`.env`を自動読み込みしません。手動実行時は、shellに環境変数を読み込んでから起動します。

```bash
set -a
source .env
set +a
uv run --frozen --extra scraping python -m src
```

### 2. 定期実行する（推奨: systemd timer）

Linuxではcronよりsystemd timerを推奨します。ログを`journalctl`で確認でき、PC停止中に予定時刻を過ぎても`Persistent=true`で次回起動時に実行できます。

`~/.config/systemd/user/econ-cal-sync.service`を作成します。

```ini
[Unit]
Description=Sync economic calendar to Google Calendar

[Service]
Type=oneshot
WorkingDirectory=/absolute/path/to/econ-cal-sync
EnvironmentFile=/absolute/path/to/econ-cal-sync/.env
ExecStart=/absolute/path/to/uv run --frozen --extra scraping python -m src
```

`command -v uv`で`uv`の絶対パスを確認し、上記のパスを置き換えてください。

`~/.config/systemd/user/econ-cal-sync.timer`を作成します。

```ini
[Unit]
Description=Run econ-cal-sync every Monday

[Timer]
OnCalendar=Mon *-*-* 07:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

登録して手動テストします。

```bash
systemctl --user daemon-reload
systemctl --user enable --now econ-cal-sync.timer
systemctl --user start econ-cal-sync.service
journalctl --user -u econ-cal-sync.service -n 100 --no-pager
systemctl --user list-timers econ-cal-sync.timer
```

ログアウト中もユーザーサービスを動かす場合は、必要に応じて次を実行します。

```bash
loginctl enable-linger "$USER"
```

### cronを使う場合

cronでも実行できますが、パスと環境変数を明示する必要があります。

```cron
0 7 * * 1 cd /absolute/path/to/econ-cal-sync && /bin/bash -c 'set -a; source .env; set +a; /absolute/path/to/uv run --frozen --extra scraping python -m src' >> /absolute/path/to/econ-cal-sync/sync.log 2>&1
```

systemd timerとcronはどちらか一方だけを登録してください。GitHub Actions側の定期実行も残す場合、同じ時刻に二重実行しないよう注意してください。

---

## カスタマイズ

`src/sync.py` の先頭付近にある定数を編集します：

```python
# 対象国の通貨コード（ForexFactory は通貨コードで国を識別します）
TARGET_COUNTRIES = {"USD", "JPY"}

# 最低重要度（1=低, 2=中, 3=高）
IMPORTANCE_MIN = 3

# 何週間先まで取得するか
FETCH_WEEKS = 4
```

新しい国を追加する場合は、`COUNTRY_FLAG` にも対応するフラグ絵文字を追加してください。

---

## 新しいデータソースの追加方法

1. `src/fetchers/my_source.py` に `BaseFetcher` を継承したクラスを作成します。
2. `name` プロパティと `fetch()` メソッドを実装し、`EconomicEvent`（`src/models.py` 定義）のリストを返すようにします。
3. `src/fetchers/__init__.py` に登録します：
   ```python
   from .my_source import MySourceFetcher
   _FETCHERS["my_source"] = MySourceFetcher
   ```
4. ワークフローで `EVENT_SOURCE=my_source` を設定します。

---

## プロジェクト構成

```
src/
├── __init__.py
├── __main__.py          # python -m src エントリポイント
├── sync.py              # メイン同期ロジック（データソース非依存）
├── models.py            # EconomicEvent データクラス
└── fetchers/
    ├── __init__.py      # フェッチャーレジストリ & get_fetcher()
    ├── base.py          # BaseFetcher 抽象基底クラス
    ├── forexfactory.py
    └── fmp.py
```
