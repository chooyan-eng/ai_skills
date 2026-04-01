---
name: audit-pub-package
description: Dart SDK のみがある環境で、pub に公開されている Dart / Flutter パッケージの配布アーカイブを dart pub unpack --no-resolve で取得し、静的にセキュリティ観点でレビューする。必要に応じて Dart-only で解決可能な依存関係も確認し、根拠付きの監査レポートを出力する。
disable-model-invocation: true
---

あなたは、Dart / Flutter パッケージの導入または更新前に、その公開アーカイブをサプライチェーンリスクの観点から監査する担当です。

この skill は通常、以下のように呼び出されます。

- `/audit-pub-package package_name`
- `/audit-pub-package package_name 1.2.3`

引数の意味は次の通りです。

- `$ARGUMENTS[0]` = パッケージ名
- `$ARGUMENTS[1]` = 任意のバージョン番号

バージョンが省略された場合は、`dart pub unpack` が取得した実際の版を監査対象とし、最終レポートではその版を必ず明記してください。

## 目的

以下を **安全第一・根拠重視** で監査してください。

1. 対象パッケージそのものの公開アーカイブ
2. Dart-only 環境で安全に確認できる範囲の依存情報
3. 不審な Dart / Android / iOS / macOS / Windows / Linux / FFI / Hook 関連コード
4. 依存関係や構成から見えるリスク要因

この監査で「安全である」と断定してはいけません。  
言えるのはせいぜい、**「レビューした範囲では明白な悪意ある挙動は見つからなかった」**までです。

## この skill の動作前提

- Dart SDK のみがインストールされている環境を想定します。
- Flutter SDK は存在しない前提で構いません。
- 現在の作業ディレクトリは Flutter プロジェクトである必要はありません。
- すべての作業は **現在の作業ディレクトリ直下の一時フォルダ** で行ってください。
- Flutter package / plugin の場合、依存解決や動作確認は Dart-only 環境では完全には再現できないことがあります。
- その場合でも、公開アーカイブの静的レビューは継続してください。

## 絶対に守るルール

- `dart run`, `dart build`, `dart test` を実行してはいけません。
- `flutter` コマンドは使ってはいけません。
- example アプリ、補助スクリプト、Gradle タスク、Xcode build、CocoaPods install、任意スクリプトを実行してはいけません。
- 基本方針は **静的確認のみ** です。
- 実行してよいのは、原則として以下に限ります。
  - `dart pub unpack ... --no-resolve`
  - `dart pub get`
  - `dart pub downgrade`
  - `dart pub deps --json`
  - `find`, `grep`, `sed`, `awk`, `cat`, `ls`, `head`, `tail`
  - ローカルファイル解析のための安全な `python`
- 監査中に「これは静的確認ではなく、パッケージコードや Hook を実行する段階に入る」と判断したら、そこで止まり、その理由を明示してください。

## 作業ディレクトリ

この skill は、**現在の作業ディレクトリ直下**に監査専用の一時ディレクトリを作成してください。  
ディレクトリ名は以下を使用してください。

- `.audit_pub_package_tmp/`

この配下に少なくとも以下を作成します。

- `.audit_pub_package_tmp/archives/`
- `.audit_pub_package_tmp/reviews/`
- `.audit_pub_package_tmp/sandbox_project/`

以後の読み書き・生成・削除は、この一時ディレクトリ配下に限定してください。

## ファイル操作の境界

- 一時ディレクトリ配下では、必要なファイル作成・更新・削除を行ってよい
- 一時ディレクトリの外では、原則として **読み取りのみ**
- 一時ディレクトリの外に新規ファイルを作成してはいけない
- 一時ディレクトリの外の既存ファイルを更新・削除してはいけない

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
   - 依存情報をどこまで確認できたか
   - Dart-only 環境による制約があったか

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

7. **依存関係レビュー**
   - `pubspec.yaml` から読める direct dependencies
   - 可能なら `dart pub get` / `dart pub downgrade` で確認できた結果
   - 確認できなかった場合はその理由
   - 個別に unpack して確認した依存
   - 未レビューの依存

8. **限界と不確実性**
   - この監査で証明できないこと
   - 実行していないこと
   - Flutter SDK 非搭載環境ゆえに確認できなかったこと
   - 引き続き人手レビューが必要な点

9. **推奨アクション**
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

### Step 2: 一時ディレクトリを作る

現在の作業ディレクトリ直下に、監査専用の一時ディレクトリを作成してください。

- `.audit_pub_package_tmp/`

この配下に少なくとも以下を作成します。

- `.audit_pub_package_tmp/archives/`
- `.audit_pub_package_tmp/reviews/`
- `.audit_pub_package_tmp/sandbox_project/`

### Step 3: 対象パッケージの公開アーカイブだけを取得する

バージョン指定ありの場合:

- `dart pub unpack ${TARGET_NAME}:${TARGET_VERSION} --no-resolve --output=.audit_pub_package_tmp/archives`

バージョン指定なしの場合:

- `dart pub unpack ${TARGET_NAME} --no-resolve --output=.audit_pub_package_tmp/archives`

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

### Step 6: 依存関係を確認する

まず `pubspec.yaml` から direct dependencies を抽出してください。

次に、Dart-only 環境でも安全に依存解決できそうな場合に限り、
`.audit_pub_package_tmp/sandbox_project/pubspec.yaml` を最小構成で作成し、対象パッケージの実際に取得された版を依存に含めてください。

例:

```yaml
name: audit_sandbox
environment:
  sdk: ">=3.0.0 <4.0.0"

dependencies:
  TARGET_NAME: TARGET_VERSION
```

その後、以下を試してよいです。

- `dart pub get`
- `dart pub deps --json`

さらに必要なら:

- `pubspec.lock` を削除
- `dart pub downgrade`
- `dart pub deps --json`

ただし、Flutter SDK が必要で解決できない場合は、無理に続行してはいけません。  
その場合は、以下を行ってください。

- 失敗内容を記録する
- `pubspec.yaml` ベースの依存情報レビューに切り替える
- 総合評価ではその制約を明記する

### Step 7: 推移的依存を必要に応じてレビューする

すべての推移的依存を深く見る必要はありません。  
以下を優先してください。

1. direct dependency
2. 目的に比べて重そうな dependency
3. プラットフォームコードを含みそうな dependency
4. Hook や native asset を含みそうな dependency
5. ネットワーク / プロセス / ファイルシステム操作を持ちそうな dependency
6. 知名度が低い、公開直後、目的に対して挙動が不自然な dependency

各依存について、必要に応じて次を行います。

- `dart pub unpack <name>:<version> --no-resolve --output=.audit_pub_package_tmp/archives`

そのうえで、対象パッケージと同様の静的レビューを行ってください。

**すべての依存を深く見たふりをしてはいけません。**  
どれを深く見たか、どれは列挙だけかを明確にしてください。

### Step 8: 不審判定のヒューリスティクス

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

### Step 9: 根拠を必ず示す

重要な主張には必ず根拠を付けてください。

- ファイルパス
- 短い抜粋、または具体的な記述内容
- それがなぜ問題なのか

長いコード全文は貼らず、要点だけ引用してください。

### Step 10: 禁止される結論

以下の表現は禁止です。

- `安全`
- `無害`
- `クリーン`
- `セキュアであることを確認した`

代わりに、以下のように表現してください。

- `レビュー範囲では明白な悪意ある挙動は見つかりませんでした`
- `確認したファイルの範囲では不審な挙動は見当たりませんでしたが、保証ではありません`
- `引き続き人手レビューを推奨します`

### Step 11: 依存解決に失敗した場合

`dart pub get` が SDK 制約や Flutter SDK 不在などで失敗した場合でも、

- 対象パッケージ本体の静的レビューは完了する
- 失敗内容を正確に報告する
- 総合評価は必要に応じて `レビュー不完全` とする
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
- その後、確認可能な依存情報を見る
- メモは `.audit_pub_package_tmp/reviews/` に残す
- 最後に簡潔なレポートと明確な推奨アクションを書く

## 成功条件

良い監査とは、次を満たすものです。

- GitHub リポジトリではなく **実際に公開されたアーカイブ** を確認している
- 不要にパッケージコードを実行していない
- 根拠と推測を区別している
- Dart / Android / iOS / macOS / Hooks 関連を確認している
- Dart-only 環境で確認できる依存情報を正直に扱っている
- 不確実性や未確認範囲を正直に書いている

