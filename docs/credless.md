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

## アクター切替 (`--as` / `--seed`) — service account 名義

個人名義 (hajime) のほかに、GitLab の service account (`yashiro-worker` /
`yashiro-reviewer` / `yashiro-supervisor` 等) を名乗って `glab` / `git` を
使い分ける仕組み (行為者分離。設計正本は館の
`report/design-agent-actors`)。対象は `glab` / `glab-rotate` / `ssh-load` /
`git` の4コマンドで、`--as <actor>` はどれもコマンド名直後のフラグ。

- actor 名は `[a-z0-9-]+`(1文字以上)。それ以外は即エラー
- 個人名義の宣言 (`tree.*.template.GLAB_CONFIG` / `GLAB_ROTATE_VAULT` /
  `SSH_LOAD_VAULT` 等) には一切触れない — `--as` 無しの挙動は変わらない
- actor の秘密は store の名前空間に置く (ツリーの `store://` — SPEC §10.7)。
  論理レイアウト:

  ```
  agents/<actor>/gitlab-pat/<host>/token       PAT (glab-rotate --seed で種蒔き)
  agents/<actor>/ssh-signing/<host>/key + /pub 署名鍵ペア (ssh-load --seed で種蒔き)
  agents/<actor>/identity/name|email|username  git commit の名義 (git --seed で種蒔き)
  ```

- `haj store put` はツリーのコマンドの中でしか動かない (素のシェルから
  `store://` は叩けない) ので、初回投入は各コマンドの **`--seed` の口**
  (stdin → store) を使う。値は echo しない — 履歴に残らない・複数行が
  成立するファイルリダイレクト (`< <ファイル>`) か、端末で `read` して
  パイプする形にする
- **`--seed` は既存値があると拒否する** (fail-fast)。上書き
  (失効時の再種蒔き等) は `--force` を付ける。`ssh-load --seed` の
  key/pub、`git --seed` の name/email/username は、いずれか1つでも
  既存なら `--force` 無しでは全体を拒否する (片肺状態からでも
  `--force` で全部を再投入できる)
- 対象ホストの列挙は `GLAB_AS_HOSTS` (空白区切り。`glab` / `glab-rotate --as` /
  `ssh-load` / `git` が使う)。未設定なら既存の `GLAB_ROTATE_HOSTS` を流用する:

  ```sh
  haj config set tree.haj-credless.env.GLAB_AS_HOSTS 'gitlab.example.com'
  ```

- 種蒔きの順序 (1 actor・1 host ぶん):

  ```sh
  haj glab-rotate --seed --as worker gitlab.example.com < <PATファイル>
  haj ssh-load    --seed --as worker gitlab.example.com < <秘密鍵ファイル>
  printf 'name=yashiro-worker\nemail=worker@example.com\nusername=yashiro-worker\n' \
    | haj git --seed --as worker
  ```

  (すでに値がある場合は `--seed --force` で入れ直す)。以後は
  `haj glab --as worker ...` / `haj glab-rotate --as worker` /
  `haj ssh-load --as worker` / `haj git --as worker <gitの引数...>`。

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

### actor 名義 (`--as`)

```sh
haj glab --as worker mr create ...
```

テンプレート宣言は使わない — `haj secret tmpdir glab-<actor>` に
config.yml を**コマンド内で**生成し、`GLAB_AS_HOSTS` (→ `GLAB_ROTATE_HOSTS`)
の各ホストの PAT を `agents/<actor>/gitlab-pat/<host>/token` から引く
(HTTPS 固定。上のアクター切替の節を参照)。環境の `GLAB_CONFIG_DIR` は
**無視する** — 無印の「env が既にあればそれが勝つ」規約はここには適用
しない。actor 用の config は毎回必ず作り直す。

### 期限切れ前のローテーション (glab-rotate)

GitLab 16.10+ の self-rotate API (`POST /personal_access_tokens/self/rotate`)
で PAT を回し、bao の種を入れ替える。config.yml は `haj glab` が毎回
レンダリングし直すので、bao さえ更新されれば他にやることは無い。

マルチホスト前提: 対象は `GLAB_ROTATE_HOSTS` (空白区切り) で列挙し、
各ホストの PAT は `<GLAB_ROTATE_VAULT>/<ホスト>` から読む (tpl の
`users/<自分>/gitlab-pat/<ホスト>` 規約と同じ)。基底パスは config-init が
提案するので、ホスト一覧だけ手動 (実効値は `haj env glab-rotate`):

```sh
haj config set tree.haj-credless.env.GLAB_ROTATE_HOSTS 'gitlab.example.com gitlab.other.com'
haj config set tree.haj-credless.env.GLAB_ROTATE_VAULT users/<自分>/gitlab-pat
```

使い方 (ホストを引数に与えればそのホストだけ。無指定は全ホスト):

```sh
haj glab-rotate --check                      # 全ホストの期限・スコープの確認のみ
haj glab-rotate                              # 残り GLAB_ROTATE_MARGIN 日 (既定7) を切っていたら回す (冪等)
haj glab-rotate --force gitlab.example.com   # そのホストだけ今すぐ回す
```

ホストごとに独立して処理し、途中で失敗しても残りを続けて最後に非0で終わる。

- 新トークンの寿命は `GLAB_ROTATE_TTL` 日 (既定 30)。無印は冪等なので
  cron やログイン時フックに置ける。
- 前提スコープ: `api` (GitLab 17.9+ なら rotate 専用の `self_rotate` でも可)。
- rotate は生きたトークンでしか呼べない。**切れた後は Web UI で再発行**して
  `bao kv put` で種蒔きし直す (この道具は期限前に回すためのもの)。
- 旧トークンは rotate の瞬間に失効する。新トークンは応答 → tmpfs 退避 →
  bao 書き戻し → 退避削除の順で扱い、書き戻しに失敗したときは退避パスと
  復旧コマンドを案内する (ロックアウト防止)。
- rotate 済みの旧トークンを使うと GitLab は再利用検知で**新トークンごと**
  ファミリー失効させる。`haj glab` は毎回レンダリングするので通常は無縁だが、
  `GLAB_CONFIG_DIR` を export したまま生かしている古いシェルからは叩かないこと。

### actor 名義 (`--as` / `--seed`)

margin/ttl/退避/独立処理の意味論は無印と同一で、PAT の読み書き先だけ
`agents/<actor>/gitlab-pat/<host>/token`(store)に切り替わる。
`GLAB_ROTATE_VAULT` は使わない。ホスト解決は
**引数 → `GLAB_AS_HOSTS` → `GLAB_ROTATE_HOSTS`** の順 (無印は引数 →
`GLAB_ROTATE_HOSTS` のまま)。

```sh
haj glab-rotate --seed --as worker gitlab.example.com < <PATファイル>   # 初回投入 (1ホストのみ)
haj glab-rotate --as worker                                             # 以後の self-rotate (冪等)
haj glab-rotate --as worker --check gitlab.example.com
```

store への書き戻しに失敗したとき (旧トークンは既に失効・新トークンは
tmpfs に退避済み) の復旧も `--seed --force` の口を使う (素のシェルから
`haj store put` は叩けないため):

```sh
haj glab-rotate --seed --force --as worker gitlab.example.com < <退避ファイル>
```

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

### actor 名義 (`--as` / `--seed`)

```sh
haj ssh-load --seed --as worker gitlab.example.com < <秘密鍵ファイル>   # 種蒔き (key + pub を導出)
haj ssh-load --as worker                                                 # ホスト省略時は GLAB_AS_HOSTS の先頭
haj ssh-load --as worker gitlab.other.com
```

鍵は `agents/<actor>/ssh-signing/<host>` の `key` (秘密鍵) / `pub` (公開鍵)。
`--seed` は stdin から OpenSSH 秘密鍵を1本読み、`ssh-keygen -y` で公開鍵を
導出して両方を store に格納する (`ssh-keygen -y` はファイルでしか読めない
ため tmpfs に一時実体化し、使用後に消す)。`--as` (無印load) は `key` を
パイプで `ssh-add` に渡すだけでディスクを経由しない。

## git — actor 名義で commit・push する

`haj git` は `--as` 専用のコマンド (個人名義の「無印」は無い — 個人の git は
そのまま素の `git` を使う)。commit の author/email・push の資格情報
(HTTPS+PAT)・commit 署名 (ssh) を `-c` でその場だけ差し替えて `exec git` する
(常駐する設定変更は無い)。

```sh
haj git --seed --as worker <<'EOF'
name=yashiro-worker
email=worker@example.com
username=yashiro-worker
EOF

haj git --as worker push origin HEAD          # SSH remote のままでも HTTPS+PAT で押せる
haj git --as worker commit -m '...'            # signed commit (ssh 署名鍵は先に ssh-load --as で agent へ)
```

`--seed` は name/email/username の**3つすべて**が揃わないと何も書かない
(部分成功を作らない)。既存の identity があれば `--force` 無しには拒否する。

- **credential**: 先頭に `-c credential.helper=`(空値)を注入して既存の
  helper 連鎖をいったんリセットしてから、インラインの `!` helper を足す
  (`~/.gitconfig` 等の他の helper が誤って併用されるのを防ぐ)。この
  helper は `get` にだけ応答し、username/password を渡す (ディスクに
  書かない。値は環境変数経由なので子プロセスの `/proc/<pid>/environ`
  からは見える — SPEC §10.11 と同じ限界)
- **URL 書き換え**: `url.https://<host>/.insteadOf` を `ssh://git@<host>/` と
  `git@<host>:` の両形に張るので、remote が SSH のままでも HTTPS+PAT で通る
- **署名**: `user.signingkey` は store の公開鍵を `key::` 直値で渡す
  (ディスク非経由)。秘密鍵は `haj ssh-load --as` で agent 側に積んでおく
  (このコマンドは公開鍵しか読まない)
- host は env `GIT_AS_HOST`、無ければ `GLAB_AS_HOSTS` → `GLAB_ROTATE_HOSTS`
  の先頭。identity (`name`/`email`/`username`) と gitlab-pat・ssh-signing の
  `pub` は全部揃っていないと動かない (揃え方は上のアクター切替の節)

## kubectl は見送り

exec-creds 式の kubeconfig (実行時にプラグインが資格情報を取得する形) を使えば
ローカルに資格情報が留まらないため、このツリーでラップする必要が無い
(kubeconfig の配布は組織ごとの事情なので、各社ツリーの領分)。