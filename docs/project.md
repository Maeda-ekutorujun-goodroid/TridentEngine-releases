# プロジェクト管理とビルド

Trident Hub、プロジェクト設定、ゲームのエクスポート、ディレクトリ構成、ビルド方法。

[← ドキュメント一覧へ戻る](index.md)

## Trident Hubとプロジェクト

`TridentHub.exe`はUnity Hubに相当する軽量なプロジェクトランチャーです。

- プロジェクト名と保存場所を指定して新規作成
- 3D、2Dのプロジェクトテンプレート
- 既存のTridentプロジェクトを追加
- 最近使ったプロジェクトを一覧表示
- 一覧からの除外（プロジェクトファイル自体は削除しません）
- 作成または選択したプロジェクトを`TridentEditor.exe`で起動
- プロジェクト単位のC++ Game Moduleビルド、Hot Reload、ゲーム書き出し

新規プロジェクトには`assets/scenes/Main.scene.json`、`.trident/project.json`、
描画に必要な組み込みShader（`assets/shaders/`のLit・Environmentと
「新規カスタムShader」の雛形）、プロジェクト用`.gitignore`とREADMEが
生成されます。空のフォルダーやサンプルは作られないので、`scripts`や
`textures`などは必要になったときにAsset Browserから自由に作成して
ください。
履歴は`%LOCALAPPDATA%/TridentEngine/hub.json`へ保存されます。

Hubは起動時にバックグラウンドで配布リポジトリ
（`TridentEngine-releases`）の最新版を確認し、新しいエンジンが
公開されていればタイトル下へ通知バナーを表示します
（「ダウンロードページを開く」「このバージョンをスキップ」）。
オフライン時や配布リポジトリが未作成の間は何も表示されません。

### エディターが落ちたとき（セーフモード）

C++スクリプトの不具合でエディターごと落ちてしまうことがあります。
その場合、次にプロジェクトを開くと選択ダイアログが表示されます。

- **通常どおり開く** — もう一度試します
- **セーフモードで開く** — **C++スクリプトを読み込まずに**起動します。
  シーンの編集や保存はできるので、原因のスクリプトを直したり、
  問題のコンポーネントを外したりできます
- **開かない** — 何もせず終了します

セーフモード中は上部に警告が出て、隣の「通常モードで開き直す」で
いつでも戻れます。Trident Hubの「セーフモードで開く」ボタン、または
`--safe`を付けた起動でも同じ状態になります。

```powershell
.\TridentEditor.exe --safe --project "C:\Projects\MyGame"
```

落ちた原因は`.trident/Crashes`に診断テキストとミニダンプとして
保存されています。

### 新しいエンジンで既存プロジェクトを開く

プロジェクトは作成時にエンジンのシェーダー（`assets/shaders/`）を
コピーして持ちます。エンジンを新しくすると内部のデータ配置が変わる
ことがあるため、**エディターはプロジェクトを開くときに組み込み
シェーダーを現在のエンジンのものへ自動で更新します**（欠けていれば
復元し、更新したことはConsoleへ表示します）。

自分でシェーダーを編集していた場合は、上書き前に`<名前>.bak`として
残します。改造内容を活かしたいときは、`.bak`と新しいファイルを見比べて
必要な変更を入れ直してください。適用済みのエンジンバージョンは
`.trident/project.json`の`engineVersion`に記録されます。

Hubを使わず、エディターを直接起動することもできます。

```powershell
.\build\TridentEditor.exe --project "C:\Projects\MyGame"
```

引数を省略した場合は、TridentEngineリポジトリ自体を既定プロジェクトとして開きます。
既存プロジェクトには`assets`フォルダーと`.trident/project.json`が必要です。

GPUが無い・正しく動かない環境（仮想マシン、リモートデスクトップなど）では、
`--warp`を付けるとGPUを使わずCPUラスタライザ（WARP）で起動できます。
描画は遅くなりますが、エディター操作や動作確認には十分です
（エクスポートしたゲームのexeでも同じフラグが使えます）。

```powershell
.\build\TridentEditor.exe --warp --project "C:\Projects\MyGame"
```

## プロジェクト設定

エディターの「ファイル」→「プロジェクト設定...」から次を編集できます。

- ゲーム名（エクスポートしたexe名とウィンドウタイトルになります）
- ゲームアイコン（assets内の.png/.jpg/.ico。Export時にexeへ埋め込み）
- 初期ウィンドウ解像度
- 起動シーン
- グラフィック品質プリセット
- 描画スケール、Shadow解像度／Cascade上限
- Bloom、FXAA、Fog、VSync、FPS上限
- テクスチャのランタイムBC1/BC3圧縮
- Point／Spot Lightの描画上限
- GameObjectタグの一覧（InspectorのTag欄の候補になります）
- 入力Actionとキーボード／ゲームパッドBinding
- 外部スクリプトエディター（`.cpp`／`.hlsl`を開くときに使うエディター）

タグを登録すると、InspectorのTag欄がドロップダウン選択になり、
Scene／Prefab読み込み時に未登録タグの使用をConsoleへ警告します。
一覧が空の間は従来通り検査なしで動作します。

「スクリプト」カテゴリーでは、Asset Browserで`.cpp`や`.hlsl`をダブルクリック
したときに開くエディターを選べます（UnityのExternal Script Editorに相当）。
「新規C++ Script」で作成した直後に開くのも同じエディターです。このPCの
Visual Studio Code（Insidersを含む）と、vswhere経由で見つかるVisual Studio
（Community／Professional／Enterprise、プレリリース版を含む）を自動検出して
一覧へ並べます。見つからない場合や別のエディターを使いたい場合は「参照...」から
実行ファイルを直接指定できます。「システムの既定（ファイルの関連付け）」を選ぶと
従来通りWindowsの関連付けで開きます。指定した実行ファイルが存在しないまま保存すると
エラーになります。

設定は`.trident/project.json`へ保存されます。起動シーンは直接入力、シーン一覧、
または「現在のシーン」ボタンから選択できます。Exportすると配布用
`TridentGame.json`へ変換され、ゲームはウィンドウ作成前にゲーム名と解像度を、
シーン読込前に起動シーンを反映します。

スクリプトエディターは`scriptEditorPath`としてこのPC上の絶対パスで保存され、
ゲームアイコンと同じく配布用`TridentGame.json`には含まれません。
`.trident/project.json`はGit管理対象なので、別のPCでは同じパスにエディターが
無いことがあります。その場合はプロジェクト設定から選び直してください。

プリセットの既定値は次の通りです。

| 品質 | 描画スケール | Shadow | Cascade | Bloom／FXAA／Fog | テクスチャ圧縮 |
|---|---:|---:|---:|---|---|
| Low | 0.65 | 無効 | 1 | 無効 | 有効 |
| Medium | 0.85 | 1024 | 2 | 有効 | 有効 |
| High | 1.00 | 2048 | 3 | 有効 | 無効 |
| Ultra | 1.00 | 4096 | 4 | 有効 | 無効 |

テクスチャ圧縮を有効にすると、PNG/JPG等の読み込み時にBC1（不透明）／BC3
（アルファ付き）へランタイム圧縮し、VRAM使用量をおよそ1/8〜1/4へ削減します。
切り替えは次に読み込まれるテクスチャから反映されます（DDSは元の形式のまま）。

個別項目を変更するとCustomになります。設定は保存直後にScene View／Game Viewへ反映され、
Export後のゲームでも`TridentGame.json`から同じ値が読み込まれます。
C++から実行中に切り替えることもできます。

```cpp
application.Graphics().ApplyQualityPreset(
    Trident::GraphicsQualityPreset::Medium);

auto settings = application.Graphics().Settings();
settings.vSyncEnabled = false;
settings.targetFrameRate = 144; // 0は無制限
settings.renderScale = 0.75f;
settings.preset = Trident::GraphicsQualityPreset::Custom;
application.Graphics().SetGraphicsSettings(settings);
```

「パフォーマンス」タブでは、FPS、Frame ms、CPU＋Present時間、直近120フレームの
グラフ、固定物理Step数・補間率・補間中Rigidbody数、Collider／Broad Phase／Contact数、描画カリング数を
確認できます。タブを閉じた場合は「ウィンドウ」→「パフォーマンス」から再表示できます。

## ゲームをエクスポート

エディターの「ファイル」→「ゲームをエクスポート...」から出力先を指定できます。
現在のシーンが保存され、プロジェクト設定で指定した起動シーンとともに次の配布物が
生成されます。

```text
TridentGame/
├─ <ゲーム名>.exe
├─ TridentRuntime.dll
├─ TridentGameModule.dll
├─ xaudio2_9redist.dll
├─ vcruntime140.dll などVC++ランタイム
├─ TridentGame.json
└─ assets.tpak
```

実行ファイルはプロジェクト設定のゲーム名になります（`\ / : * ? " < > |`
などファイル名に使えない文字は`_`へ置換）。ゲームアイコンを設定していれば
exeへ埋め込まれ、Explorerのファイルアイコンとウィンドウのタイトルバー
アイコンが自分のゲームのものになります。

ダイアログの「配布用ZIPも作成」にチェックを入れると、出力フォルダーの隣に
そのまま配れる`<フォルダー名>.zip`も作成されます。

`TridentGame.json`には起動シーンが保存されるため、Sandboxの固定シーンに依存せず、
現在編集中のシーンからゲームを開始できます。再Exportではステージングへ全ファイルを
作成してから既存パッケージを置き換えるため、コピー途中の不完全な配布物を残しません。

CMakeからは次のコマンドで`build-release/export/TridentGame`へ出力できます。

```powershell
cmake --build build-release --target TridentGamePackage
```

配布版エンジンからエクスポートするとVC++ランタイムDLLが同梱されるため、
渡した相手のPCに追加のインストールは不要です（ソースからビルドした
エディターで、エディターの隣にランタイムDLLが無い場合は従来どおり
Microsoft Visual C++ Redistributable 2022以降が必要です）。

エクスポートしたゲームでは**F1キー**でデバッグオーバーレイを表示できます。
FPS・フレーム時間、GameObject数と描画カリング統計、Collider数、
直近の警告/エラーログが画面左上に出るため、「書き出したら動かない」ときの
原因調査に使えます（もう一度F1で閉じます）。

## ディレクトリ

```text
TridentEngine/
├─ .trident/               プロジェクト／エディター設定
├─ assets/                 シーン、Prefab、各種ゲームアセット
├─ cmake/                  依存関係の検出
├─ samples/Game/           エディターなしゲームサンプル
├─ samples/GameModule/     ゲーム固有C++ DLLサンプル
├─ samples/Sandbox/        エディター付きサンプル
├─ src/Trident/
│  ├─ Animation/           Animation Clipとキーフレーム補間
│  ├─ Assets/              アセット読み込みとキャッシュ
│  ├─ Audio/               DirectXTK AudioEngineとWAV／OGGキャッシュ
│  ├─ Components/          Camera、Mesh、Spriteなど
│  ├─ Core/                ウィンドウとゲームループ
│  ├─ Editor/              Dear ImGuiエディター
│  ├─ Graphics/            Direct3D 11管理
│  ├─ Input/               Action Mappingと入力状態
│  ├─ Scene/               GameObject、階層、JSON
│  └─ Scripting/           Game Module ABIとHot Reload
├─ tests/                  シーン往復テスト
└─ third_party/            DirectXTK、Dear ImGui、nlohmann/json
```

## ビルド

Visual StudioのDeveloper Command Prompt、または`vcvars64.bat`を実行したターミナルでビルドします。

CMake Presetを使う場合:

```powershell
cmake --preset windows-debug
cmake --build --preset windows-debug
ctest --preset windows-debug
```

```powershell
cmake -S . -B build -G "NMake Makefiles" -DCMAKE_BUILD_TYPE=Debug
cmake --build build
.\build\TridentHub.exe
.\build\TridentEditor.exe --project "."
.\build\TridentGame.exe
```

Releaseビルド:

```powershell
cmake -S . -B build-release -G "NMake Makefiles" -DCMAKE_BUILD_TYPE=Release
cmake --build build-release
```

テスト:

```powershell
ctest --test-dir build --output-on-failure
```

配布用ZIP:

```powershell
cmake --preset windows-release
cmake --build --preset windows-package
```

ZIPは`out/build/windows-release`以下へ生成されます。ZIPにはゲーム制作へ
必要なものだけが入り、サンプルゲームやテストは含まれません（PDBは
リリース時に別の`TridentEngine-symbols`ZIPとして配布されます）。
ルートの`build-info.json`にはエンジン版とGitビルド識別子が含まれます。
クラッシュ診断にも同じ識別子が記録され、実行ファイル横、または
プロジェクトの`.trident/Crashes`へ保存されます。

Visual Studioでは「ローカル フォルダーを開く」で、このフォルダーをCMakeプロジェクトとして扱えます。

## リリース（配布パッケージの公開）

バージョンは日付ベース（CalVer）で`YYYY.M.D`形式です（例: `2026.7.31`。
同じ日に複数回リリースする場合は`2026.7.31.1`のように4つ目の数字を
増やします）。`v2026.7.31`形式のタグをプッシュすると、GitHub Actionsが
ビルド・全テスト・パッケージングを実行し、GitHub Releasesへ配布物を
自動で添付します（テストが1つでも失敗するとリリースは作成されません）。

配布はソース本体を公開せずに行えます。リリースは開発リポジトリに加えて
**配布専用の公開リポジトリ
[TridentEngine-releases](https://github.com/Timiratz/TridentEngine-releases)**
へも自動で作成され、友達への配布URLとHubのアップデート通知は
こちらを参照します。

自動公開には開発リポジトリのActionsシークレット`DIST_REPO_TOKEN`が
必要です（未設定の間、公開ステップは自動スキップ）。gh CLIから
次の1コマンドで登録できます。

```bash
gh secret set DIST_REPO_TOKEN --repo Timiratz/TridentEngine --body "$(gh auth token)"
```

より権限を絞りたい場合は、Fine-grained Personal Access Token
（Repository access: `TridentEngine-releases`のみ / Contents:
Read and write）を作成して同じ名前で登録してください。

- `TridentEngine-<version>-windows-x64.zip` — エンジン一式
- `TridentEngine-symbols-windows-x64.zip` — クラッシュ解析用PDB

リリースノートは`CHANGELOG.md`の該当バージョン節から自動で抜き出されます。

手順:

1. `CHANGELOG.md`の`Unreleased`節を新しいバージョン節（`## YYYY.M.D`）へ確定する
2. `CMakeLists.txt`の`project(TridentEngine VERSION YYYY.M.D ...)`を更新する
3. タグを作成してプッシュする

```bash
git tag v2026.7.31
git push origin v2026.7.31
```
