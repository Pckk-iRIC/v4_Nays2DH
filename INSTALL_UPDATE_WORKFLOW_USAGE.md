# install更新Workflowの使い方

## 概要

`.github/workflows/update-install.yml` は、手動実行でソルバーをビルドし、署名した `install/Nays2DH.exe` をデフォルトブランチへ反映するWorkflowです。

このWorkflowは `online_update_v4` などの外部リポジトリを操作しません。デフォルトブランチへの反映後に既存の `build.yml` が発火することはあります。

## 実行方法

1. GitHubリポジトリの **Actions** を開く
2. **install更新（手動・署名）** を選択する
3. **Run workflow** を押す
4. ブランチやコンパイラの入力は指定せずに実行する

Workflowは常にビルド・署名・`install`更新まで実行します。`build`や`apply`などの切り替えフラグはありません。

## コンパイラの扱い

ビルド方法はリポジトリの `make.bat` を正とします。Workflowは実行開始時に有効なコマンド行から `ifort` または `ifx` を検出します。

- 現在の `make.bat` は `ifort` を呼び出すため、ifort環境を構築します
- 将来 `make.bat` を `ifx` 呼び出しへ変更した場合は、ifx環境の構築経路へ分岐します
- コメント行に書かれたコンパイラ名は検出しません
- ifortとifxの両方、またはどちらも検出できない場合はビルドを中止します
- 検出後も `make.bat` は書き換えず、引数なしで実行します

したがって、コンパイラを変更するときはWorkflowの実行入力ではなく、検証済みの `make.bat` を更新します。

## Secret

今回、指定された `sign_binary.py` から次のリポジトリSecretを `gh secret set` で登録済みです。

- `AZURE_KEY_VAULT_URI`
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_CERT_NAME`

署名SecretはリポジトリSecretとして利用します。Environmentの作成や承認設定は必要ありません。

`AZURE_EXPECTED_SIGNER_THUMBPRINT` は指定スクリプトに値が含まれていなかったため、現時点では未登録です。未登録の場合でも署名ステータスとタイムスタンプ証明書は検証し、Thumbprintの期待値比較だけを警告付きで省略します。証明書のThumbprintが確定したら、次のコマンドで追加登録してください。

```powershell
gh secret set AZURE_EXPECTED_SIGNER_THUMBPRINT --repo Pckk-iRIC/v4_Nays2DH
```

`Downloads\sign_binary.py` は認証情報がハードコードされたローカルファイルのため、リポジトリへ追加しません。Workflowでは同じAzureSignToolの引数構成を使用し、認証情報はSecretsから環境変数として渡します。

## 反映の流れ

1. デフォルトブランチをcheckout
2. `make.bat` のコンパイラを検出
3. 対応するIntel FortranとVisual Studio環境を構築
4. `make.bat` を実行
5. `install/Nays2DH.exe` にAzureSignToolで署名
6. 署名とタイムスタンプを検証
7. `install` 以外を含めずにコミット
8. デフォルトブランチへ直接プッシュ
9. 直接プッシュが失敗した場合だけ更新ブランチを作成し、Pull Requestを作成

Pull Requestは自動マージしません。内容を確認して手動でマージしてください。マージ後のpushで既存の `build.yml` が発火します。

## 権限設定

Workflowでは次の権限を宣言しています。

- `contents: write`：デフォルトブランチまたは更新ブランチへのpush
- `pull-requests: write`：直接push失敗時のPull Request作成

デフォルトブランチが保護されている場合は直接pushが拒否され、フォールバック経路が使われます。ActionsによるPull Request作成がリポジトリ設定で許可されている必要があります。

## 失敗時の確認

- コンパイラ検出に失敗した場合：`make.bat` の有効なコンパイラ呼び出しを確認する
- ビルドに失敗した場合：`make.bat` が要求するIntel FortranとVisual Studioの組み合わせを確認する
- 署名Secretエラーの場合：リポジトリSecret名と値の設定を確認する
- 署名検証エラーの場合：Artifactの `署名検証レポート` を確認する
- 直接pushとPR作成の両方に失敗した場合：`contents: write`、`pull-requests: write`、ActionsのPR作成許可を確認する

同じ実行を再度行う場合は、Actions画面の **Re-run jobs** を使用します。
