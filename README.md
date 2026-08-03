# TridentEngine 配布物

2D/3DゲームエンジンTridentEngineのダウンロードページです。
ここにはビルド済みの配布物だけを置いています。

## ダウンロード

最新版は [Releases](../../releases) からどうぞ。

- `TridentEngine-<バージョン>-windows-x64.zip` — エンジン一式
  （TridentHub / TridentEditor / Runtime / サンプルアセット / 日本語ドキュメント）
- `TridentEngine-symbols-windows-x64.zip` — クラッシュ解析用PDB（通常は不要）

## 使い方

1. エンジン一式のZipを展開します
2. `TridentHub.exe` を起動します（VC++ランタイム同梱、追加インストール不要）
3. 初回起動でSmartScreenの警告が出た場合は「詳細情報」→「実行」を選びます
4. 使い方はZip内の `docs/index.md`（日本語ドキュメント）を参照してください

バージョンは日付形式です（例: `2026.7.31` = 2026年7月31日版）。

# TridentEngine

TridentEngineは、C++20・DirectX 11・DirectXTK11で作る軽量な2D/3Dゲームエンジンです。
Unity風の`GameObject + Component`を採用し、同じシーン内で3Dオブジェクトと2Dスプライトを扱えます。

現在のバージョンは、ゲーム用`TridentRuntime.dll`とDear ImGui製の軽量エディターを
分離しています。同じJSONシーンをエディター付きSandboxとエディターなしGameの両方で
実行できます。

**はじめての方は[入門チュートリアル](docs/getting-started.md)からどうぞ。**
プロジェクト作成からスクリプトで動かしてエクスポートするまでを30分で追えます。

## ダウンロード

ビルド済みのエンジン一式は、配布専用の公開リポジトリ
[TridentEngine-releases](https://github.com/Timiratz/TridentEngine-releases/releases)から
ダウンロードできます（ソースコード本体は非公開のまま、配布物だけを
公開しています）。ZIPを展開して`TridentHub.exe`を起動すれば、
プロジェクト作成からゲームのエクスポートまでビルド不要で使えます
（VC++ランタイムDLL同梱のため、追加のインストールは不要です）。

- `TridentEngine-<version>-windows-x64.zip` — エンジン一式
  （TridentHub／TridentEditor／Runtime／日本語docs。サンプルやテストは
  含まない、ゲーム制作に必要なものだけの構成）
- `TridentEngine-symbols-windows-x64.zip` — クラッシュ解析用PDB

## ドキュメント

使い方は[docs/のドキュメント一覧](docs/index.md)へ機能別にまとめています。

- [入門チュートリアル](docs/getting-started.md) — 最初のゲームを30分で
- [エディター](docs/editor.md) / [C++スクリプティング](docs/scripting.md)
- [グラフィックス](docs/graphics.md) / [物理](docs/physics.md) / [UIと2D](docs/ui-2d.md) / [Animation](docs/animation.md)
- [SceneとPrefab](docs/scenes.md) / [オーディオ](docs/audio.md) / [入力](docs/input.md) / [NavMesh](docs/navigation.md)
- [プロジェクト管理とビルド](docs/project.md)

## 実装済み

- Win32アプリケーションとゲームループ
- ゲーム用`TridentRuntime.dll`とエディターライブラリの分離
- エディターなし`TridentGame.exe`
- Direct3D 11デバイス、スワップチェーン、深度バッファ
- ウィンドウリサイズ
- `Scene` / `GameObject` / `Component` / `Transform`
- 親子GameObjectとワールドTransform
- 透視投影カメラ
- DirectXTK `GeometricPrimitive`による3D描画
- Scene環境光、Directional／Point／Spot Light
- Trident Lit HLSLによる距離減衰・スポットコーン照明
- Cook-Torrance GGXによる物理ベースシェーディング（roughness／metallic）
- キューブマップSkyboxとIBL（環境反射・環境拡散）
- IBLのGGX事前フィルタ（粗さに正確な環境反射＋コサイン畳み込み放射照度）
- MeshRendererのGPUインスタンシング（同一形状・同一マテリアルの自動一括描画）
- Directional Lightの最大4分割・256～8192pxカスケードシャドウ
- スポットライトのシャドウマップ（同時4灯）
- ポイントライトのキューブシャドウマップ（同時1灯）
- 全シャドウのPCFソフトシャドウ（カスケード/スポット3x3、ポイント5タップ）
- Low／Medium／High／Ultra／Customグラフィック品質
- 描画スケール、FXAA、VSync、Bloom／Fog／Shadow品質制御
- LitMaterialによるアルベド・法線マップ・粗さ設定
- `.material.json`によるLitMaterialの共有・再利用
- 静的ModelRendererの共通Lit描画と影の受光
- glTF 2.0（`.gltf`／`.glb`）モデル読み込み
- FBX（ASCII／Binary）モデル読み込み
- glTFスキン、最大72ボーンのGPUスキニング
- 複数アニメーションクリップ、STEP／Linear／CubicSpline補間
- ModelRendererのクリップ選択、速度、ループ、自動再生、スクラブ
- ModelRendererのAnimator Controller、ボーン姿勢クロスフェード
- DirectXTK `SpriteBatch`による2D描画
- DirectXTK `Keyboard`による全キー入力（A～Z、0～9、F1～F12、編集キー、修飾キー）
- マウス入力（左/右/中/X1/X2ボタン、縦横ホイール、カーソル移動量）
- Win32イベント蓄積による1フレーム未満のキー・クリック検出
- DirectXTK `GamePad`によるゲームパッド入力
- Input Action MappingとPressed／Held／Released状態
- GameObjectのTagとFindGameObjectByName／FindGameObjectsByTag検索API
- プロジェクト設定のタグ一覧（Inspectorはドロップダウン選択、未登録タグへ警告）
- シーン全体のFindComponentOfType／FindComponentsOfType検索
- 階層内検索（GetComponentInChildren/InParent、FindChildパス指定）
- `Time::SetTimeScale`によるポーズ・スロー・倍速再生
- ScriptのAwake／Start／OnEnable／OnDisable／OnDestroy／LateUpdateライフサイクル
- ScriptのInvoke／InvokeRepeating／CancelInvokeタイマー
- Scriptコルーチン（co_await WaitForSeconds/WaitUntil/WaitWhile/次フレーム）
- 名前付きイベントバス（Script::On/Emit、UI Buttonのクリックイベント発行）
- isTrigger接触のOnTriggerEnter／Stay／Exit振り分け
- 親GameObjectの無効化が子孫へ伝わるIsActiveInHierarchy
- JSON AnimationClipとTransformキーフレーム補間
- TransformAnimatorのループ、速度、自動再生、プレビュー
- Animation TimelineでのClip作成・Keyframe記録・編集
- Animator ControllerのState、Trigger／Exit Time遷移、クロスフェード
- DirectXTK `AudioEngine`と内蔵VorbisデコーダーによるWAV／OGG再生とキャッシュ
- AudioSourceのBGM／効果音、音量、ピッチ、パン、ループ、開始時再生
- 長尺BGMのストリーミング再生（圧縮データのみ保持し逐次デコード）
- 効果音／BGMバスとマスター音量のミキサー
- AudioListenerとAudioSourceの3D定位・線形距離減衰
- WIC/DDSテクスチャをキャッシュするAssetManager
- JSONシーン保存・読み込み
- `.prefab.json`によるGameObject階層の保存・再利用
- Prefabインスタンスのリンク、Override検出、Apply／Revert
- HierarchyとInspector
- 保存可能なドッキング式エディターレイアウト
- Transformと組み込みコンポーネントの編集
- GameObjectの作成と階層単位の削除
- Componentの追加と削除
- Hierarchyのドラッグ＆ドロップ親子変更
- HierarchyのGameObject／シーンルート／空白右クリックメニュー
- 最大64状態のUndo／Redo
- GameObject階層の複製、コピー、カット、ペースト
- Asset Browserによる一覧／サムネイルグリッド表示
- Asset Browserのフォルダーツリーとパンくずナビゲーション
- Asset Browserからのフォルダー作成・名前変更・空フォルダー削除
- Asset Browserでのファイル名変更・安全な削除・ドラッグ移動
- Asset Browserのファイル／フォルダー右クリックメニュー
- Asset Browserからのシーン切り替え
- SpriteRendererへのテクスチャ割り当てと差し替え
- SpriteRendererのソース矩形（アトラス/スプライトシートの部分表示）
- Sprite Animatorによるスプライトシートのコマ送りアニメーション
- SpriteRenderer／ParticleSystemの個別カスタムHLSLシェーダー（8パラメータ、ホットリロード）
- ScreenEffect（任意HLSLの全画面ポストエフェクト）
- 2D/UIをBloom・トーンマッピング・FXAA適用後に合成（UIの色と輪郭を維持）
- Scene ViewとGame Viewの独立タブ
- RenderTargetを使ったオフスクリーン描画
- Main Cameraとは独立したScene Camera
- ImGuizmoによる移動・回転・拡縮操作
- ギズモ操作のUndo／Redo
- Windows日本語フォントの自動検出
- 日本語UIとIME入力
- 日本語GameObject名・シーン・アセットパスのUTF-8保存
- CMO／SDKMESH／VBO／glTF／GLB／FBX形式の3Dモデル読み込み
- ModelRendererとコンポーネント単位の独立Effect
- 2Dボックスコライダー
- 3Dボックスコライダー
- Mesh Collider（glTF／GLB／FBXの三角形メッシュ、BVH、三角形単位の精密Raycast）
- 2D円コライダー（CircleCollider2D）と円×円／円×ボックスの2D衝突
- Rigidbody、質量、重心、角速度、トルク、回転／位置Constraint、Continuous CCD
- Continuousモードの実スイープによる高速移動時のトンネリング防止
- カプセル×ボックスの2点接触マニフォールド
- PhysicsMaterialの合成モード選択（平均／幾何平均／乗算／最小／最大）
- Raycast／SphereCast／BoxCast／CapsuleCast／Overlap系クエリ
- Fixed／Hinge／Spring Joint
- AABB衝突判定と最小押し戻し
- 2D／3D空間ハッシュBroad Phaseと候補ペア重複除去
- Collision Enter／Stay／Exitイベント
- トリガー判定
- DirectWriteによるゲーム内日本語テキスト
- TextRendererと日本語文字テクスチャキャッシュ
- UI Canvasの基準解像度スケーリング
- UI Rect Transformのアンカー、Pivot、Stretch
- UI ButtonのHover／Pressed／Clickと日本語ラベル
- UI Imageの単色／テクスチャ／9-slice描画
- UI Toggleのオン・オフ切り替えと変更通知
- UI Sliderのドラッグ操作、範囲設定、整数モード
- UI Input FieldのフォーカスとIME確定文字を含むテキスト入力
- UI Layout Groupによる子Rectの水平／垂直自動整列
- UI Scroll View（ホイール／ドラッグスクロール、シザー矩形クリッピング、スクロールバー）
- Scene Managerの遅延切り替え・再読み込み・失敗時ロールバック
- Sceneの非同期読み込み、進捗API、キャンセル、標準Loading画面
- ワーカースレッドでのテクスチャ非同期作成（WICデコード〜GPUテクスチャまで）
- テクスチャのランタイムBC1/BC3圧縮（品質設定で切り替え、VRAM削減）
- GPUの無い環境（VM・リモートデスクトップ）でのWARP自動フォールバック
- バックバッファのピクセル読み出しAPI（CaptureBackBuffer、回帰テスト用）
- NavMeshの視線判定によるパス平滑化（string pulling）
- NavMeshAgentの複数サーフェス自動選択（高さの近いフロアを優先）
- LOD Group、Frustum Culling、CPU Occlusion Culling、描画統計
- JobSystem（ワーカースレッドプール＋ParallelFor）とフラスタム判定の並列化
- ConsoleのInfo／Warning／Error、検索、停止、ファイル出力
- Audio／Game Module／Editor／Simulation／Render別CPUプロファイラー
- GPU区間計測（タイムスタンプクエリ。影/3D/スカイ/2D/ポスト処理別）
- PerformanceパネルからのプロファイルJSON出力
- 未処理例外時の診断テキストとWindows minidump出力
- ゲーム内デバッグオーバーレイ（F1でFPS・描画/物理統計・直近の警告ログ）
- Scene ViewでのColliderデバッグ表示
- Scene Viewでのレイ/AABBクリック選択と選択枠
- Scene ViewのXZグリッド、間隔／範囲設定
- 移動／回転／拡縮のTransformスナップ
- Scene ViewのView Cubeと選択オブジェクトへのフォーカス
- Cameraコンポーネントの視錐台ギズモ
- Scene Viewの透視投影／正投影切り替え
- Scene Cameraの移動速度・高速倍率・回転／ズーム感度設定
- プロジェクト単位のエディター設定保存・自動復元
- 名前付きエディター設定プリセットの作成・切り替え
- 32レイヤーの衝突レイヤー／マスク
- Play／Stop時の編集状態復元
- 新規シーン、上書き保存、名前を付けて保存
- ファイルメニューまたはCtrl+Sによるシーン保存
- エディターとCMakeからのゲームExport
- 起動シーンを保持する`TridentGame.json`
- ステージングと旧出力退避による安全な再Export
- ゲーム名・初期解像度・起動シーンのプロジェクト設定
- プロジェクト単位のキー／ゲームパッドAction割り当て
- `.trident/project.json`からゲームウィンドウへの設定反映
- シーン保存／読み込みの自動テスト
- ゲームExporterの自動テスト
- Windows CIによるReleaseビルド、CTest、配布物生成

## 次の開発候補

（宣言済みの開発候補はすべて実装済みです。次のアイデアを募集中！）

## ライセンス

TridentEngine本体はMITライセンスです。DirectXTK、Dear ImGui、nlohmann/jsonも
MITライセンスです。XAudio2 Redistributableには
Microsoftのライセンスが適用されます。詳細は`THIRD_PARTY_NOTICES.md`を参照してください。
