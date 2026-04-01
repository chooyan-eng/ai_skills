---
name: audit-flutter-package
description: Dart / Flutter パッケージを導入・更新する前に、公開アーカイブを dart pub unpack --no-resolve で取得して静的レビューし、さらに隔離した一時プロジェクトで解決される推移的依存についても再帰的に確認し、根拠付きの監査レポートを出力する。
disable-model-invocation: true
---

あなたは、Dart / Flutter パッケージの導入または更新前に、その公開物をサプライチェーンリスクの観点から監査する担当です。

この skill は通常、以下のように呼び出されます。

- `/audit-flutter-package package_name`
- `/audit-flutter-package package_name 1.2.3`

引数の意味は次の通りです。

- `$ARGUMENTS[0]` = パッケージ名
- `$ARGUMENTS[1]` = 任意のバージョン番号

バージョンが省略された場合は、`dart pub unpack` が取得した実際の版を監査対象とし、**最終レポートでは実際に取得された版を必ず明記**してください。

## 目的

以下を **安全第一・根拠重視** で監査してください。

1. 対象パッケージそのものの公開アーカイブ
2. 隔離した一時プロジェクトにおいて、その対象パッケージ版を依存に追加したときに解決される推移的依存パッケージ群

この監査で「安全である」と断定してはいけません。  
言えるのはせいぜい、**「レビューした範囲では明白な悪意ある挙動は見つからなかった」**までです。

## 絶対に守るルール

- `dart run`, `flutter run`, `dart build`, `flutter build`, `dart test`, `flutter test` を実行してはいけません。
- パッケージ内の example アプリ、補助スクリプト、Gradle タスク、Xcode ビルド、CocoaPods install、任意スクリプトを実行してはいけません。
- 基本方針は **静的確認のみ** です。
- 実行してよいのは、原則として以下に限ります。
  - `dart pub unpack ... --no-resolve`
  - `dart pub get`
  - `dart pub downgrade`
  - `dart pub deps --json`
  - `find`, `grep`, `sed`, `awk`, `cat`, `ls`, `head`, `tail`
  - ローカルファイル解析のための安全な `python`
- 一時プロジェクトは必ず破棄可能な隔離環境に作ってください。
- 監査中に「これは静的確認ではなく、パッケージコードや Hook を実行する段階に入る」と判断したら、そこで止まり、その理由を明示してください。

## 監査の考え方

これはコード品質レビューではなく、**サプライチェーン監査**です。

以下のようなシグナルを優先的に見てください。

- シークレット、環境変数、SSH 鍵、Git 認証情報、CI 変数、署名ファイル、keystore、provisioning profile、Firebase 認証情報、API トークンなどを読むコード
- 外部通信、テレメトリ、解析、アップロード、ダウンロード、remote config、隠しエンドポイント
- 動的コード読み込み、バイナリのダウンロード、シェル実行、プロセス起動
- 予想外のネイティブコードや FFI
- 難読化、極端に読みにくい generated code、意図的に読解を妨げる構造
- Hook、およびビルド時挙動に影響するコード
- Android の Gradle スクリプト、Manifest、Service、Receiver、Permission、JNI、外部ダウンロード
- iOS/macOS の Podspec、xcconfig、build phase、shell script、dynamic framework、entitlement、URL/network 関連コード
- パッケージの本来の目的と実際の挙動の不一致
- 小さな patch version に不釣り合いな大きな変更
- platform channel、reflection、広範なファイル探索、clipboard、スクリーンショット、accessibility API、WebView bridge、ランタイム割り込みなどの不審な利用

一般的によくある書き方であることを理由に安全とみなしてはいけません。  
また、不審な記述が見当たらないことをもって安全の証明としてはいけません。

## 最終出力の形式

最終回答は、必ず次の構成で出力してください。

1. **監査対象**
   - パッケージ名
   - 要求されたバージョン
   - 実際に取得されたバージョン
   - 監査日時

2. **総合評価**
   - 次のいずれかを使う
     - `レビュー範囲では明白な悪意ある挙動は見つからなかった`
     - `不審`
     - `高リスク`
     - `レビュー不完全`
   - 重要な理由を 3〜8 個の箇条書きで示す

3. **何をレビューしたか**
   - 対象パッケージのどのパスを確認したか
   - Android コードの有無
   - iOS / macOS コードの有無
   - Hook 関連ファイルや Hook を示唆する構成の有無
   - レビューした推移的依存の数
   - 高い側 / 低い側の両方の依存解決を確認したか

4. **発見事項**
   - `Critical`
   - `High`
   - `Medium`
   - `Low`
   - `Informational`
   各項目について次を含める
   - タイトル
   - 重大度
   - 根拠となるファイルパス
   - それが問題になる理由
   - 明確に悪意があるのか、不審なのか、妥当な可能性はあるがレビュー継続が必要なのか

5. **Hooks レビュー**
   - Hook 関連機能がありそうか
   - 注意すべきファイルと記述
   - build 時ダウンロード、native asset 生成、build/run/test 時の副作用の可能性
   - この監査では Hook を実行していないことを明記する

6. **プラットフォーム別レビュー**
   - Android
   - iOS / macOS
   - FFI / ネイティブバイナリ

7. **推移的依存レビュー**
   - `dart pub get` で解決された正確なバージョン
   - `dart pub downgrade` で解決された正確なバージョン
   - それらの間で変化したパッケージ
   - 個別に unpack して確認した推移的依存
   - 時間や範囲の都合で未レビューの依存

8. **限界と不確実性**
   - この監査で証明できないこと
   - 実行していないこと
   - 引き続き人手レビューが必要な点

9. **推奨アクション**
   - 次のいずれか
     - `導入してよい`
     - `人手レビュー後に導入`
     - `保留`
     - `避ける`
   - その理由を短く書く

## 手順

### Step 1: 引数を解釈する

以下のように扱ってください。

- `TARGET_NAME = $ARGUMENTS[0]`
- `TARGET_VERSION = $ARGUMENTS[1]`（指定がある場合のみ）

パッケージ名が指定されていない場合は、期待する呼び出し形式を説明して停止してください。

### Step 2: プロジェクトルート直下に隔離ワークスペースを作る

現在のプロジェクトルート直下に、監査専用の一時ディレクトリを作成してください。  
ディレクトリ名は衝突しにくい名前を使ってください。例:

- `.audit_pub_package_tmp/`
- `.claude_audit_workspace/`

このディレクトリ配下に少なくとも以下を作成します。

- `.audit_pub_package_tmp/archives/`
- `.audit_pub_package_tmp/reviews/`
- `.audit_pub_package_tmp/sandbox_project/`

以後の読み書き・生成・削除は、**この一時ディレクトリ配下に限定**してください。

### ファイル操作の境界

- 一時ディレクトリ配下では、必要なファイル作成・更新・削除を行ってよい
- 一時ディレクトリの外では、原則として **読み取りのみ**
- 一時ディレクトリの外に新規ファイルを作成してはいけない
- 一時ディレクトリの外の既存ファイルを更新・削除してはいけない
- 例外は、監査対象としてユーザーが明示したファイルの読み取りのみ

### 作業ディレクトリの原則

コマンド実行時は、必ず以下のいずれかを working directory としてください。

- プロジェクトルート直下の一時ディレクトリ
- その配下の `archives/`
- その配下の `sandbox_project/`

`/tmp` やホームディレクトリなど、プロジェクト外の場所を作業ディレクトリにしてはいけません。

### Step 3: 対象パッケージの公開アーカイブだけを取得する

バージョン指定ありの場合:

- `dart pub unpack ${TARGET_NAME}:${TARGET_VERSION} --no-resolve --output=archives`

バージョン指定なしの場合:

- `dart pub unpack ${TARGET_NAME} --no-resolve --output=archives`

作成されたディレクトリ名や `pubspec.yaml` から、**実際に取得された版**を特定してください。

### Step 4: 対象パッケージを静的確認する

以下のファイルやディレクトリを、存在する限り確認してください。

- `pubspec.yaml`
- `pubspec.lock`（存在すれば）
- `README*`
- `CHANGELOG*`
- `LICENSE*`
- `analysis_options.yaml`
- `lib/**`
- `bin/**`
- `tool/**`
- `hooks/**`
- `build.dart`
- `build_hooks/`
- `native_assets/`
- `android/**`
- `ios/**`
- `macos/**`
- `linux/**`
- `windows/**`
- `ffi/**`
- `src/**`
- generated file や同梱バイナリ

特に以下を探してください。

- ネットワーク通信やハードコードされたエンドポイント
- プロセス実行
- 想定外に広いファイルシステム探索
- シークレット収集
- ダウンロード処理
- shell script
- 動的読み込み
- reflection / introspection
- native code、podspec、gradle、独自 build ロジック
- パッケージの説明と噛み合わない処理

不審なコードについては、以下を明示してください。

- 本番用コードなのか
- test/example 限定なのか
- build 時のみなのか
- 死んだコードなのか
- まだ判断できないのか

### Step 5: Hook が見つからなくても Hooks セクションは必ず書く

必ず Hooks セクションを出力してください。

Hook や native asset 関連の構成が見つかった場合は、次を説明してください。

- どのファイルからそう判断したか
- build/run/test 時に何ができそうか
- なぜそれが重要か

見つからなかった場合でも、次の文言を含めてください。

- `レビューした公開アーカイブ内では、明白な Hook 関連ファイルや native asset 構成は見当たりませんでした。`

そのうえで、**見当たらないことは不存在の証明ではない**と補足してください。

### Step 6: 隔離した依存解決用プロジェクトを作る

`sandbox_project/pubspec.yaml` を最小構成で作成し、対象パッケージの実際に取得された版を依存に含めてください。

`sandbox_project/pubspec.yaml` は、プロジェクトルート直下の一時ディレクトリ配下、
`PROJECT_ROOT/.audit_pub_package_tmp/sandbox_project/` に作成してください。

依存解決で生成される `pubspec.lock`、`.dart_tool/`、ログ、JSON 出力もすべてこのディレクトリ配下に保存してください。

例:

```yaml
name: audit_sandbox
environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  TARGET_NAME: TARGET_VERSION
```

Flutter plugin などで依存解決上の調整が必要なら行ってよいですが、build や run に進んではいけません。

その後、`sandbox_project` 内で次を実行します。

- `dart pub get`
- `dart pub deps --json`

これで、**高い側の依存解決結果**を保存してください。

### Step 7: 低い側の依存解決も確認する

同じ `sandbox_project` 内で、

- 生成された lockfile を削除する
- `dart pub downgrade`
- `dart pub deps --json`

を実行し、**低い側の依存解決結果**も保存してください。

比較対象は次です。

- 対象パッケージ自身の版
- direct dependency
- transitive dependency
- 高い側と低い側で版が変わったパッケージ

### Step 8: 推移的依存を再帰的にレビューする

再帰的にレビューしますが、優先順位をつけてください。

優先順位:

1. 前回既知の lockfile と比べて新たに現れたパッケージ
2. 今回の更新で版が変わったパッケージ
3. プラットフォームコードを含むパッケージ
4. Hook や native asset を含みそうなパッケージ
5. ネットワーク / プロセス / ファイルシステム操作を持つパッケージ
6. 知名度が低い、公開直後、目的に対して挙動が不自然なパッケージ

各依存について、必要に応じて次を行います。

- `dart pub unpack <name>:<version> --no-resolve --output=archives`
- 対象パッケージと同様の静的レビュー
- 重大な問題がない限り、まとめは簡潔でよい

**すべての推移的依存を深く見たふりをしてはいけません。**  
どれを深く見たか、どれは列挙だけかを明確にしてください。

### Step 9: 不審判定のヒューリスティクス

次のような組み合わせがあれば重大度を上げてください。

- 単純な UI / utility パッケージのはずなのに、アップロード・ダウンロード・隠し通信・shell 実行・ネイティブバイナリを含む
- patch version なのに networking、telemetry、build script、難読化、新しいプラットフォームコードが増えている
- build 時に外部から成果物を取得する
- plugin コードが広い Android 権限を追加している
- podspec や gradle が不自然な remote fetch を行っている
- コメントや関数名が debugging、exfiltration、persistence、self-update を示唆する

逆に、次のような場合は重大度を下げてもよいです。

- test 専用で、本番 / build パスから到達しない
- 通信処理がパッケージの本来目的そのものであり、README にも説明がある
- generated file が通常のツール由来で、目的とも整合している

ただし、その場合でも記録は残してください。

### Step 10: 根拠を必ず示す

重要な主張には必ず根拠を付けてください。

- ファイルパス
- 短い抜粋、または具体的な記述内容
- それがなぜ問題なのか

長いコード全文は貼らず、要点だけ引用してください。

### Step 11: 禁止される結論

以下の表現は禁止です。

- `安全`
- `無害`
- `クリーン`
- `セキュアであることを確認した`

代わりに、以下のように表現してください。

- `レビュー範囲では明白な悪意ある挙動は見つかりませんでした`
- `確認したファイルの範囲では不審な挙動は見当たりませんでしたが、保証ではありません`
- `引き続き人手レビューを推奨します`

### Step 12: 依存解決に失敗した場合

`dart pub get` が SDK 制約やプラットフォーム制約で失敗した場合でも、

- 対象パッケージ本体の静的レビューは完了する
- 失敗内容を正確に報告する
- 総合評価は `レビュー不完全` とする
- 安全に続行するために何が必要か、可能なら説明する

## 実際に探すべきパターン

レビュー時には、以下のキーワードや API を積極的に検索してください。

- `http`, `dio`, `socket`, `WebSocket`, `InternetAddress`, `RawSocket`
- `Process.run`, `Process.start`, `shell`, `bash`, `sh`, `cmd`
- `File`, `Directory`, `Platform.environment`
- `MethodChannel`, `EventChannel`, `BasicMessageChannel`
- `DynamicLibrary.open`, `ffi`
- `curl`, `wget`, `git clone`, `pod`, `gradle`, `exec`
- `URLSession`, `NSURL`, `OkHttp`, `HttpURLConnection`
- `keystore`, `provisioning`, `sign`, `credentials`, `token`, `secret`, `key`
- `hook`, `native_asset`, `build`, `download`, `bundle`

また、以下も確認してください。

- 追加された permission
- background service
- broadcast receiver
- custom URL scheme
- clipboard access
- WebView bridge
- analytics SDK
- バイナリ blob や圧縮アーカイブ

## 推奨する進め方

- 最初に全体の棚卸しをする
- 次に不審ポイントへ絞る
- その後、解決された推移的依存を確認する
- メモは `reviews/` に残す
- 最後に簡潔なレポートと明確な推奨アクションを書く

## 成功条件

良い監査とは、次を満たすものです。

- GitHub リポジトリではなく **実際に公開されたアーカイブ** を確認している
- 不要にパッケージコードを実行していない
- 根拠と推測を区別している
- Dart / Android / iOS / macOS / Hooks 関連をカバーしている
- 実際に解決された推移的依存を確認している
- 不確実性や未確認範囲を正直に書いている
