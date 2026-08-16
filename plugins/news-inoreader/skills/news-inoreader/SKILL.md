---
name: news-inoreader
description: Inoreader API を使用してニュース記事を検索・管理する。「Inoreaderでニュースを検索して」「未読記事を確認して」「フィードを購読・整理して」と言われたときに使う。
allowed-tools: mcp__plugin_news_inoreader__*
---

# Inoreader ニュース検索スキル

Inoreader API を使用してニュース記事の検索およびフィードの管理を行うスキル。

## 前提条件（Linux環境）

`auth_login` はOAuthトークンをOSのキーチェーンに保存する仕組みのため、Linux環境では `libsecret-tools`（`secret-tool` コマンド）が事前にインストールされている必要がある。未インストールの場合、認証ログインが失敗する。

```
sudo apt install -y libsecret-tools
```

sudo実行にはパスワードの対話入力が必要なため、エージェントが代行実行できない場合はユーザー自身の端末（`!` プレフィックス）で実行してもらう。

### WSL2特有の注意: secret-toolはコマンドがあるだけでは動かない

`isKeychainAvailable` 相当のチェックは `secret-tool` コマンドの**存在**しか見ておらず、実際にSecret Service（`org.freedesktop.secrets`）を提供するデーモン（gnome-keyring等）が起動しているかは確認していない。WSL2はデスクトップログインを経由しないため、`gnome-keyring-daemon` が既定では起動していないことが多い。

この状態だと `auth_login` はブラウザでの認可自体は成功し「Authentication Successful!」まで表示されるが、その直後のトークン保存（`secret-tool store`）で必ず失敗して落ちる。事前に以下で疎通確認しておくとよい。

```
secret-tool store --label=test service test account test <<< testpass
secret-tool lookup service test account test   # testpass が返れば正常
secret-tool clear service test account test     # 後片付け
```

`secret-tool: The name org.freedesktop.secrets was not provided by any .service files` と出た場合はkeyringデーモンが動いていない。

```
sudo apt-get install -y gnome-keyring dbus-x11
```

インストール後、シェルセッションごとにkeyringのunlockが必要（`.bashrc`/`.zshrc`に追記しておくと便利）。

```bash
if [ -z "$GNOME_KEYRING_CONTROL" ]; then
  eval "$(printf '\n' | gnome-keyring-daemon --unlock --start --components=secrets)"
  export GNOME_KEYRING_CONTROL
  export SSH_AUTH_SOCK
fi
```

sudoが絡む部分はエージェントが代行できないため、ユーザー自身の端末（`!` プレフィックス）で実行してもらう。

## 認証（`auth_login`）時の注意

`auth_login` が発行する認可URLの `redirect_uri` はサーバー側の実装によって `127.0.0.1` または `localhost` になりうる。Inoreader側のOAuth検証は `http://127.0.0.1` 形式の `redirect_uri` を拒否するため、`127.0.0.1` のままだと認可自体が失敗する。`inoreader-mcp-server` 側では `localhost` 固定に修正済み（コミット `fe7870e`）だが、他実装・古いバージョンを使っている場合は `127.0.0.1` を `localhost` に置き換えてブラウザで開くと成功する（ローカルのコールバックサーバーは同じポートで待ち受けているため到達する）。

なお、Windows/WSL2環境では逆に `localhost` でないと接続できない既知の問題がある（`localhost` がIPv6の `::1` に解決される一方、サーバー側がIPv4のみで待ち受けているとブラウザからの接続が失敗する）。そのため `redirect_uri` とコールバックサーバーのbind先ホスト名は両方とも `localhost` に統一しておくのが安全。

## ツール

### `search_articles`
キーワードで全フィード横断の記事を検索する。
- `query`: 検索キーワード。
- `limit`: 取得件数（デフォルト20）。

### `list_feeds`
購読中の全フィードを一覧表示する。

### `list_articles`
特定フィードの記事を一覧表示する。
- `feed_id`: フィードのID。
- `limit`: 取得件数。

### `get_article_content`
特定記事の本文全文を取得する。
- `article_id`: 記事のID。
