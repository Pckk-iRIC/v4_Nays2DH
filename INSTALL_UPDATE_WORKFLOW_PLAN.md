# install 更新 Workflow 計画

## 目的

ソルバーを手動で必ずビルドし、生成した実行ファイルを `install` ディレクトリへ反映したうえで、ソルバーリポジトリのデフォルトブランチに取り込めるようにする。

この Workflow 自体では `online_update_v4` などの外部リポジトリを扱わない。既存の `build.yml` は残し、デフォルトブランチへの更新を契機に既存 Workflow が発火することは許容する。

## スコープ

新しい Workflow の対象は次の範囲とする。

- 手動実行（`workflow_dispatch`）
- デフォルトブランチを対象としたビルド
- `make.bat` に定義された ifort ビルド環境の構築
- `make.bat` によるソルバーのビルド
- `install/Nays2DH.exe` の更新
- Azure Key Vault の証明書を使った exe 署名
- 署名の検証
- `install` だけを変更する commit と、直接 push 失敗時の Pull Request

新しい Workflow では、次の処理は行わない。

- `online_update_v4` の checkout
- Qt Installer Framework の取得
- `meta` の外部リポジトリへのコピー
- `repogen` の実行
- 外部リポジトリへの push

## 既存 Workflow との関係

現在の `.github/workflows/build.yml` は `main` / `master` への push で発火し、`online_update_v4` への反映まで行う。

そのため、新しい Workflow が `install/Nays2DH.exe` を更新した場合、直接 push が成功すればそのまま既存 Workflow が発火し、失敗時だけ更新ブランチと Pull Request を経由する。

```text
新 Workflow
  └─ install/Nays2DH.exe をビルド・署名
  └─ default branch へ直接 push
        ├─ 成功: push による既存 Workflow の発火
        └─ 失敗: 更新ブランチへ push、Pull Request を作成
              ↓ merge による push を検知
既存 build.yml
  └─ online_update_v4 へ既存の処理を実行
```

この既存 Workflow の発火は意図的に止めない。新 Workflow 自体が外部リポジトリを操作しないことと、既存 Workflow が push 後に動作することは別の責務として扱う。

現在の `config.json` は `build=false` なので、既存 `build.yml` はソルバーの再ビルドを行わず、デフォルトブランチに反映済みの `install` を利用して後続処理を行う。

## 実行モード

新しい Workflow は常にコンパイルして更新する。発火は手動実行とし、コンパイラを選択する入力は持たせない。ビルド方法はリポジトリにある検証済みの `make.bat` を正とする。

### 常時ビルド・更新

- `make.bat` が要求する ifort / Visual Studio 環境を構築する
- リポジトリの `make.bat` をそのまま実行する
- ビルド出力を `install/Nays2DH.exe` として確認する
- 署名と検証を行う
- `install` の変更だけを commit し、デフォルトブランチへの直接 push を試行する
- デフォルトブランチへの直接 push を試行し、失敗時だけ Pull Request を作成する

## `config.json` との関係

新しい Workflow は `config.json` の `build` 値を参照せず、常にコンパイルする。Workflow から `config.json` は書き換えない。

既存の `build.yml` は引き続き `config.json` の `status` / `build` を読む。現在の `build=false` は既存 Workflow にだけ適用され、新しい Workflow の常時ビルド方針とは独立している。

## コンパイラ環境

現在の [make.bat](make.bat) は `ifort` と `vs2019` を直接指定している。これは Visual Studio、Intel Fortran、リンカーの互換性を含めて検証されたビルド手順として扱う。今回の Workflow では手動入力によるコンパイラ切り替えは追加しないが、`make.bat` の検出結果に応じて ifx 用環境へ分岐できる構造にする。

`make.bat` をビルド仕様の唯一の入口とし、Workflow は次の責務だけを持つ。

- `make.bat` が要求する ifort / Intel oneAPI 環境を用意する
- `make.bat` が要求する Visual Studio のビルド環境を用意する
- `make.bat` を引数なしで実行する
- `FC`、`FORTRAN_COMPILER` などをWorkflowから上書きしない

将来 ifx へ移行する場合は、まず検証済みの `make.bat` をリポジトリ側で更新し、その後にランナー環境のバージョンを合わせる。手動実行時の入力でコンパイラを切り替える方式は採用しない。

実装時には、次も確認する。

- `where ifort` の結果
- コンパイラのバージョン
- Visual Studio のリンカーが PATH / LIB に設定されていること
- `intel64`、x64 を対象にしていること
- コンパイル前に `.obj`、`.mod`、既存 exe を削除すること
- コンパイル終了後に `install/Nays2DH.exe` が存在し、サイズが 0 ではないこと

Cabernet2D の参考 Workflow では ifx 用の `fortran-lang/setup-fortran@v1` を使用しているが、今回の Workflow では手動入力によるコンパイラ切り替え方式を採用しない。現在は ifort 用の Intel oneAPI バージョンと導入方法を固定し、将来 ifx が検出された場合の環境構築経路も YAML に用意する。

ifort 用の Intel oneAPI バージョンを明示的に固定し、`latest` や `windows-latest` の暗黙の切り替えには依存しない。

### `make.bat` からのコンパイラ検出

Workflow は `workflow_dispatch` の入力でコンパイラを指定しない。ビルド開始時に `make.bat` の有効なコマンド行を読み取り、`ifort` または `ifx` の呼び出しを検出して、対応するランナー環境を構築する。

- 現在の `make.bat` は `ifort` を検出し、ifort 用の環境を構築する
- 将来 `make.bat` が `ifx` を呼び出す形に更新された場合は、ifx 用の環境を構築できる
- `rem` / `::` などのコメント行は検出対象から除外する
- ifort と ifx の両方、またはどちらも検出できない場合は、安全側に倒してビルドを開始せず失敗させる
- 検出後も `make.bat` 自体は書き換えず、引数なしで実行する

現在の環境構築は `fortran-lang/setup-fortran@v1` の `intel-classic` 2021.10 をifort用、`intel` 2025.0をifx用に使用する。Intelの古いHPCKitダウンロードURLには依存しない。

## 署名

署名は次の順番で実行する。

```text
build
  ↓
install/Nays2DH.exe の存在・サイズ確認
  ↓
AzureSignTool で署名
  ↓
署名とタイムスタンプを検証
  ↓
install の差分を確認
  ↓
commit / Pull Request
```

今回の `install` には現時点で `Nays2DH.exe` が実行ファイルとして存在するため、初期スコープではこの exe を署名対象とする。将来 `.dll` などの PE ファイルを同梱する場合は、署名対象へ追加する。

署名方式と Secret 名は、Cabernet2D の既存 Workflow と合わせる。Secret の値はこの文書やリポジトリには記載しない。

### 共有する Secret 名

- `AZURE_KEY_VAULT_URI`
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_CERT_NAME`
- `AZURE_EXPECTED_SIGNER_THUMBPRINT`

署名処理は次の構成を踏襲する。

- `AzureSignTool` 7.0.1 を .NET global tool として導入
- `--auth-mode client-secret` を使用
- SHA-256 で署名
- RFC 3161 タイムスタンプを付与
- PowerShell の Authenticode 検証で署名状態を確認
- 署名者の Thumbprint が `AZURE_EXPECTED_SIGNER_THUMBPRINT` と一致することを確認
- 署名検証レポートを Artifact として保存する

署名用 Job は Environment による承認を使わず、登録済みのリポジトリ Secret を利用する。署名処理を実行できるWorkflow権限は `permissions` で明示する。

## 反映方法

基本経路は default branch への直接 push とする。直接 push ができない場合だけ、Pull Request 方式へフォールバックする。

1. デフォルトブランチを checkout
2. ビルド・署名・検証を実行
3. `install/Nays2DH.exe` だけを commit
4. default branch へ直接 push を試行
5. push 成功時は処理を完了し、既存 `build.yml` の発火を待つ
6. push 失敗時は同じ commit を更新ブランチへ push
7. 更新ブランチから default branch 向け Pull Request を作成
8. Pull Request を確認して手動で merge
9. merge による push で既存 `build.yml` が発火

直接 push が許可されている環境では操作を増やさず、ブランチ保護が有効な環境では Pull Request に切り替えられる。

直接 push の失敗を検知するステップは Workflow を失敗終了させず、フォールバック処理へ進める。フォールバック時には、push が部分的に成功していないことを確認してから更新ブランチを作成する。

どちらの方式でも、反映対象は `install` 配下だけに限定する。`src`、`lib`、`.github`、`config.json` などに予期しない差分がある場合は失敗させる。

## 同時実行と安全策

- `workflow_dispatch` のみで自動発火させない
- コンパイラを必ず明示的に選択する
- 更新元はデフォルトブランチに固定する
- default branch への直接 push を第一候補にする
- 直接 push の失敗時だけ更新ブランチと Pull Request を作成する
- 同じソルバーの更新は `concurrency` で直列化する
- 既存の未コミット差分があれば開始時に失敗させる
- 変更対象を `install/Nays2DH.exe` に限定する
- ビルド出力の存在・サイズを検証する
- 署名検証に失敗した場合は commit しない
- 署名済み exe と検証レポートを Artifact に保存する
- Workflow の Action は可能な限りメジャーバージョンまたは commit SHA を固定する
- 直接 push 用に `contents: write` を付与する
- フォールバック用に `pull-requests: write` も付与する
- GitHub Actions に Pull Request 作成を許可するリポジトリ設定を確認する

`GITHUB_TOKEN` の権限は Workflow の `permissions` で明示する。直接 push の可否は default branch の保護設定にも依存し、Pull Request 作成の許可は Workflow ファイルだけでなくリポジトリの Actions 設定にも依存する。

- [Workflow syntax: GITHUB_TOKEN permissions](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [Managing GitHub Actions settings](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)
- [Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

## 実装時に作成するファイル（合計2ファイル）

今回の新規作成ファイルは、この計画書を除いて次の2ファイルとする。

1. `.github/workflows/update-install.yml`
   - 手動実行専用の Workflow
   - `make.bat` のコンパイラ検出と対応環境の構築
   - ビルド、exe 署名、署名検証
   - `install` の差分限定、デフォルトブランチへの直接 push
   - 直接 push 失敗時の更新ブランチ push と Pull Request 作成
2. `INSTALL_UPDATE_WORKFLOW_USAGE.md`
   - Workflow の手動実行方法
   - `make.bat` からの ifort / ifx 検出ルール
   - 必要な Secret、権限設定
   - 直接 push と Pull Request フォールバックの動作
   - 失敗時の確認方法と再実行手順

署名・検証処理は今回の2ファイル内に定義し、Cabernet2D の既存 Workflow と同じ Secret 名、署名方式を使用する。外部リポジトリからスクリプトを実行時に取得したり、追加の署名用スクリプトを新規作成したりしない。`AZURE_EXPECTED_SIGNER_THUMBPRINT` が未登録の場合は、署名ステータスとタイムスタンプを検証し、Thumbprintの期待値比較だけを警告付きで省略する。

## 完了条件

- 引数なしで `make.bat` を実行したビルドが成功する
- `install/Nays2DH.exe` が更新される
- exe の署名とタイムスタンプを検証できる
- `AZURE_EXPECTED_SIGNER_THUMBPRINT` が登録されている場合に署名者 Thumbprint を検証できる
- `install` 以外の差分を commit しない
- default branch への直接 push が成功する、または更新ブランチへの push と Pull Request 作成へフォールバックできる
- デフォルトブランチへの反映後、既存 `build.yml` が想定どおり発火する
- 新 Workflow 自体から外部リポジトリへの操作を行わない

## 参照

- Cabernet2D の署名 Workflow: `C:\Users\yuuta.ochiai\Documents\51_mm\Cabernet2d-main\.github\workflows\release.yml`
- Cabernet2D の再署名 Workflow: `C:\Users\yuuta.ochiai\Documents\51_mm\Cabernet2d-main\.github\workflows\resign-release.yml`
- [Microsoft SignTool](https://learn.microsoft.com/en-us/windows/win32/seccrypto/signtool)
- [Intel oneAPI Fortran Compiler のリリースノート](https://www.intel.com/content/www/us/en/developer/articles/release-notes/fortran-compiler/2025.html)
