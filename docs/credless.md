# credless — 資格情報を手元に留めない CLI 群 (glab / gh / oci / aws / kc / ssh-load)

機密性が高いのに資格情報がローカルPCに放置されがちな CLI 群 (glab, gh, oci, aws,
kc, ssh) を、haj の仕組みで**手元に何も留めずに**使えるようにするツリー。
token・秘密鍵・credentials のディスク常駐を、bao 常駐 + tmpfs 一時実体化
(`haj secret file` / `template` / `tmpdir`、セッション寿命) に置き換える。

**ツリー分類の原則**: ここに居るのは**どの組織でも使える資格情報 CLI 群**
(仕組みとして汎用なもの)。会社でしか意味を持たないもの (CA・クラスタ・
ssh config 等) は各社ツリー (avap 等) へ、Mattermost 操作は mattermost ツリーへ。
会社ツリーは**名前空間公開** (`expose = namespace`) で会社文脈を呼び出しの形で
明示する (avap は `haj avap <名前>`)。こちらは名前がそのまま意味になる
(glab/oci/aws 等の固有名) ので flat のまま。

原則はどのコマンドも同じ:

- 資格情報の**宣言** (どの vault パスを引くか) はユーザー設定の
  `tree.<インストール名>.secret.*` / `.template.*` に本人が書く
  (入口は `haj tree configure <インストール名>` の提案)
- コマンドは要る瞬間に `haj secret get / file / template` で pull する。
  実体がディスクに書かれる場合も `$XDG_RUNTIME_DIR` (tmpfs) のみ —
  セッション終了で自然消滅し、掃除という概念が無い
- 環境変数が既にあればそれが勝つ (その場の明示 > 宣言)

## セットアップ

```sh
haj tree install https://github.com/AvapCoLtd/haj-credless.git
haj tree configure haj-credless          # 宣言の初期値提案 (確認して y)
haj secret list --tree haj-credless      # 宣言の目録
haj secret check --tree haj-credless     # 参照の検証 (vault には触らない)
```

## glab

`tree.<インストール名>.template.GLAB_CONFIG` に tpl のパスを宣言する
(config-init は env./secret. しか取り込めないため、これだけ手動):

```sh
haj config set tree.haj-credless.template.GLAB_CONFIG ~/.config/glab-cli/config.yml.tpl
```

tpl は素の config.yml の token 部分を vault 参照 (vault-agent template の
正準形) にしたもの。ホストごとの節の例:

```yaml
hosts:
    gitlab.example.com:
        {{- with secret "users/<自分>/gitlab-pat/gitlab.example.com" }}
        token: {{ .Data.data.token }}
        {{- end }}
        git_protocol: ssh
        api_protocol: https
```

`haj glab ...` は `haj secret tmpdir glab` (tmpfs・0700・セッション寿命) に
config.yml をレンダリングし、`GLAB_CONFIG_DIR` を据えて glab を exec する。

## gh

宣言 1 件 (config-init が提案する):

```
secret.GH_TOKEN = vault://users/<自分>/github/token
```

gh は `GH_TOKEN` 環境変数を保存済み設定より優先するので、テンプレートは不要 —
`haj gh ...` が宣言から pull して注入するだけで、`gh auth login` のトークン常駐
(`~/.config/gh/hosts.yml`) が要らなくなる。種蒔き:

```sh
bao kv put users/<自分>/github token=ghp_...
```

GitHub Enterprise は `GH_HOST` を env で渡し、`GH_ENTERPRISE_TOKEN` を同様に
宣言すればよい (環境が既にあればそれが勝つ)。

## oci

宣言 5 件 (config-init が提案する):

```
secret.OCI_CLI_USER        = vault://users/<自分>/oci/user
secret.OCI_CLI_TENANCY     = vault://users/<自分>/oci/tenancy
secret.OCI_CLI_FINGERPRINT = vault://users/<自分>/oci/fingerprint
secret.OCI_CLI_REGION      = vault://users/<自分>/oci/region
secret.OCI_KEY             = vault://users/<自分>/oci/private_key
```

`OCI_KEY` は**秘密鍵の中身そのもの**を bao に置く。`haj oci ...` が
`haj secret file OCI_KEY` で一時実体化し、`OCI_CLI_KEY_FILE` で渡す。
種蒔き:

```sh
bao kv put users/<自分>/oci \
  user=ocid1.user.oc1..xxx tenancy=ocid1.tenancy.oc1..xxx \
  fingerprint=xx:xx:... region=ap-tokyo-1 \
  private_key=@/path/to/oci_api_key.pem
# 置いたら元の ~/.oci/ の鍵ファイルは消してよい (それがこのツリーの目的)
```

## aws

宣言 2 件 (config-init が提案する):

```
secret.AWS_ACCESS_KEY_ID     = vault://users/<自分>/aws/access_key_id
secret.AWS_SECRET_ACCESS_KEY = vault://users/<自分>/aws/secret_access_key
```

種蒔き:

```sh
bao kv put users/<自分>/aws access_key_id=AKIA... secret_access_key=...
```

bao に未格納のまま `haj aws ...` を叩くと、この種蒔き手順 (実際の宣言から
導いた bao パス入り) がエラーに出る。

AWS_SESSION_TOKEN / AssumeRole / SSO のような高度な形は将来対応
(現状は静的なアクセスキーのみ)。

## kc — Keycloak Admin API CLI

Keycloak の Admin REST API を叩く CLI。kcadm.sh (Java dist が丸ごと必要) は使わず、
`glab api` と同型の curl ラッパー。認証は service account の client_credentials で、
client_secret は画面に出さず宣言から注入する。トークンは 1 実行につき 1 回だけ
取得して Bearer で使い回す。

接続先と client_secret は組織の値なので config-init は提案しない。手動で設定する:

```sh
haj config set tree.haj-credless.env.KC_URL https://keycloak.example.com/auth
haj config set tree.haj-credless.env.KC_REALM myrealm
haj config set tree.haj-credless.secret.KC_CLIENT_SECRET 'vault://<共有パス>/client_secret'
```

- **サブコマンド**:
  - `kc api <METHOD> <path> [json]` — Admin REST API 素通し (`/admin/realms/<realm>`
    起点。`/admin/…` や `/realms/…` で始めれば realm 起点を挟まず素通し)。
    GET は即実行。POST/PUT/DELETE/PATCH は `--yes` を付けたときだけ送信し、
    無しや `--dry-run` では送らず内容だけ表示する (本番 IdP 前提なので既定は安全側)。
  - 読み取り砂糖 (すべて GET): `kc users [<検索>]` / `kc user <username>` /
    `kc groups` / `kc clients [<検索>]`。
  - `kc whoami` — 今のトークンの azp と realm-management ロールを表示 (疎通・権限確認)。
  - `kc token` — アクセストークンを 1 個発行して表示 (デバッグ用、機微なので普段は使わない)。
- **ENV 化** (`haj env kc` で実効値):
  `KC_URL` (必須。旧レイアウトなら `/auth` 接頭辞込み) / `KC_REALM` (既定 master) /
  `KC_TOKEN_REALM` (クライアントが認証する realm。既定は KC_REALM — master の
  クライアントを借りるときだけ変える) / `KC_CLIENT_ID` (既定 haj-kc) /
  `KC_BAO_PATH` (任意。client_secret を置く kv パス) / `BAO_ADDR`。
  `KC_CLIENT_SECRET` は表示せず、環境 (--secret) > 宣言
  `tree.<インストール名>.secret.KC_CLIENT_SECRET` からの pull > `KC_BAO_PATH`
  自動取得 (bao セッションはコアの連鎖が確保) の三段で解決。

### 専用クライアントの作り方 (初回のみ)

日常運用ではフル admin クライアントに相乗りせず、**読み取り最小権限の専用
クライアント** (例: `haj-kc`) を対象 realm に作って使う:

1. 対象 realm に confidential + service account のクライアントを作る
   (`serviceAccountsEnabled: true` / `standardFlowEnabled: false` /
   `directAccessGrantsEnabled: false`)
2. service account に realm-management の読み取りロールを付与。最小セット:
   view-users, query-users, query-groups, view-clients, view-realm
   (403 が出た読み取りに応じて足す。書込を許すなら manage-users を追加)。
   管理コンソール (Clients > 対象 > Service account roles) からの付与が確実 —
   Admin REST でも可能だが、取り違えると権限過多になる
3. client secret を取り出して vault に置き、宣言する。ローテーションは
   `POST /clients/<uuid>/client-secret` で再生成 → vault 更新

既存の admin クライアント (master realm) を借りて 1 を REST でやるなら、
`KC_CLIENT_ID=<admin側> KC_TOKEN_REALM=master` をシェル前置で渡して
`kc api POST /clients '{...}' --yes` (haj は env を透過する)。

### 使い方

```sh
# 既定 (専用クライアント作成済みの前提。secret は宣言 or KC_BAO_PATH から自動)
haj kc users <検索>          # ユーザー一覧
haj kc user  <username>      # 1 ユーザーの詳細 + 所属グループ
haj kc groups                # グループツリー
haj kc clients [<検索>]      # クライアント一覧
haj kc whoami                # トークンの権限確認
haj kc api GET /users?max=5  # 生 API (整形なし)

# secret をその場で明示注入する場合 (宣言より手前で勝つ)
haj --secret KC_CLIENT_SECRET=vault://<共有パス>/client_secret kc users
```

## ssh-load — ssh 鍵を bao から agent に積む

秘密鍵を `~/.ssh/` に置かず bao に格納し、使うセッションで
`haj ssh-load` して agent にだけ積む (期限付き)。鍵は
bao → パイプ → `ssh-add -t` と渡り、ディスクを経由しない。

設定 (config-init が提案する。`haj env ssh-load` で実効値):

```
env.SSH_LOAD_VAULT = users/<自分>/ssh-keys       # bao の kv パス
env.SSH_LOAD_KEYS  = <フィールド名> <フィールド名...>  # 積む鍵 (空白区切り)
# SSH_LOAD_TTL は既定 8h (env で上書き可)
```

種蒔き (フィールド名 = 鍵の呼び名):

```sh
bao kv put users/<自分>/ssh-keys mykey=@/path/to/id_rsa another=@...
# 置いたら元の秘密鍵ファイルは消してよい (それがこのツリーの目的)
```

bao セッションが無ければ **haj コアの自動ログイン連鎖**が確保する
(token lookup → `secrets.vault_cert_login` の cert 認証 → `secrets.vault_login`
の OIDC。`haj docs secrets`)。ツリーは認証手順を知らない。

## kubectl は見送り

exec-creds 式の kubeconfig (実行時にプラグインが資格情報を取得する形) を使えば
ローカルに資格情報が留まらないため、このツリーでラップする必要が無い
(kubeconfig の配布は組織ごとの事情なので、各社ツリーの領分)。