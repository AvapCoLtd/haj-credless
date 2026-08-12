資格情報を手元に留めない: 鍵・PAT・アクセスキーは宣言から pull し、ディスクに常駐させない。

haj aws <awsの引数...>       AWS CLI を宣言の資格情報で起動 (credentials 非常駐)
haj glab <glabの引数...>     glab を宣言のテンプレート設定で起動 (PAT 非常駐)
haj glab-rotate              glab の PAT を self-rotate して bao を更新 (期限前に回す)
haj gh <ghの引数...>         gh を宣言のトークンで起動 (GH_TOKEN 注入・PAT 非常駐)
haj oci <ociの引数...>       OCI CLI を宣言の資格情報で起動 (鍵は一時実体化)
haj kc <サブコマンド> ...    Keycloak Admin REST API (client_credentials)
haj ssh-load                 ssh 鍵を bao から agent へ (ディスク非経由・期限付き)

アクター切替 (--as <actor>・actor 名は [a-z0-9-]+): glab/glab-rotate/ssh-load/git が
service account 名義で使える (store の agents/<actor>/... から引く。個人名義には無関係)。
種蒔きは各コマンドの --seed (stdin → store。値は echo せずファイルリダイレクトで)。
既存値があれば --seed は拒否する — 上書きは --seed --force:

haj glab --as <actor> <glabの引数...>          actor 名義で glab (PAT は store から。GLAB_CONFIG_DIR は無視)
haj glab-rotate --as <actor> [--check|--force] actor の PAT を self-rotate (store 更新)
haj glab-rotate --seed [--force] --as <actor> <ホスト> < <PATファイル>   actor の PAT を種蒔き (1行のみ)
haj ssh-load --as <actor> [<ホスト>]           actor の署名鍵を agent へ (ディスク非経由)
haj ssh-load --seed [--force] --as <actor> <ホスト> < <秘密鍵ファイル>  actor の秘密鍵を種蒔き (公開鍵も導出)
haj git --as <actor> <gitの引数...>            actor 名義で git (credential/署名を注入)
haj git --seed [--force] --as <actor>          actor の identity を種蒔き (name/email/username 3つ必須)
