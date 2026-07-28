# credless — haj ツリー

{@project haj}
{@link rel=profile} [haj profile](https://gitlab.avaper.day/avap/haj/gitlab-profile)

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

| コマンド | 置き換えるもの | 仕組み |
|---|---|---|
| `haj glab` | `~/.config/glab-cli/config.yml` の PAT 常駐 | tpl を `haj secret template` でレンダリング → `GLAB_CONFIG_DIR` (tmpfs) |
| `haj gh` | `gh auth login` の hosts.yml へのトークン常駐 | `haj secret get` で pull して `GH_TOKEN` 注入 |
| `haj oci` | `~/.oci/config` + 秘密鍵ファイル | `haj secret get` で pull、鍵は `haj secret file` で一時実体化 |
| `haj aws` | `~/.aws/credentials` | `haj secret get` で pull して env 注入 |
| `haj kc` | Keycloak Admin API の client_secret 常駐 | `haj secret get` で pull (環境 > 宣言 > bao 直の三段) |
| `haj ssh-load` | ssh 秘密鍵のディスク常駐 | bao → パイプ → `ssh-add -t` (agent の中とセッション寿命だけ) |

```sh
haj tree install https://github.com/AvapCoLtd/haj-credless.git
haj tree configure haj-credless    # 宣言一式の初期値提案 (vault のユーザー名で個人化)
```

詳細と bao への種蒔き手順は `haj docs credless`。
コマンドの書き方は `haj docs writing-commands`、配り方は `haj docs trees`。