資格情報を手元に留めない: 鍵・PAT・アクセスキーは宣言から pull し、ディスクに常駐させない。

haj aws <awsの引数...>       AWS CLI を宣言の資格情報で起動 (credentials 非常駐)
haj glab <glabの引数...>     glab を宣言のテンプレート設定で起動 (PAT 非常駐)
haj glab-rotate              glab の PAT を self-rotate して bao を更新 (期限前に回す)
haj gh <ghの引数...>         gh を宣言のトークンで起動 (GH_TOKEN 注入・PAT 非常駐)
haj oci <ociの引数...>       OCI CLI を宣言の資格情報で起動 (鍵は一時実体化)
haj kc <サブコマンド> ...    Keycloak Admin REST API (client_credentials)
haj ssh-load                 ssh 鍵を bao から agent へ (ディスク非経由・期限付き)
