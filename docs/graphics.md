# グラフィックス

3Dモデル、ライティング、Skybox / Fog / Bloom、Litマテリアル、Particle、LODと描画カリング。

[← ドキュメント一覧へ戻る](index.md)

## まず見栄えを上げる3ステップ

難しい設定を覚える前に、この3つだけで画面の印象が大きく変わります。

1. **影をつける** — Hierarchyの「太陽光」（平行光源）を選び、Inspectorで
   影を有効にします。品質プリセットがLowだと影は無効なので、
   プロジェクト設定でMedium以上にします。
2. **空・霧・発光** — Inspectorの「シーン環境」でSkybox、Fog、Bloomを
   ONにします。外部テクスチャなしで空と遠景のなじみ、光のにじみが出ます。
3. **材質感** — MeshRendererのマテリアルで「粗さ」を下げると金属や
   プラスチックのような光沢に、上げるとマットな質感になります。
   `.dds`キューブマップをSkyboxへ設定すると、粗さに応じた正確な
   映り込み（IBL）になります。

## 3Dモデル

Asset BrowserとModelRendererは次のDirectXTK形式に対応しています。

- `.cmo`
- `.sdkmesh`
- `.vbo`
- `.gltf`
- `.glb`
- `.fbx`

FBXは中間形式へ変換せず、元の`.fbx`からメッシュ、マテリアル、スキン、
アニメーションを直接読み込んで描画します。

GameObjectへModelRendererを追加し、Asset Browserでモデルを右クリックして
「モデルを割り当て」を選ぶか、Inspectorへドラッグ＆ドロップします。

### FBXの静的パーツ自動結合

アニメーションを持たないFBXでは、インポート時にボーン（スキン）の付いていない
パーツを同じマテリアルどうしで1つの頂点／インデックスバッファへ自動結合します。
DCCツールから、ボディパネル・ボルト・トリムなどが数百個の別パーツのまま書き出されて
いると、パーツ単位のドローコールと状態変更がCPU負荷の支配要因になるためです。

結合結果はConsoleの`FBX imported:`ログに`parts=717->15 (merged)`の形で出ます。
結合してもなおドローコールが64を超えるモデルには、DCCツール側でメッシュを
まとめてから書き出すことを勧める警告を出します。スキン付き、またはアニメーションを
含むモデルは結合対象外で、従来どおりの描画経路を使います。

FBX側の単位設定が実寸と食い違っている場合は、Asset Browserで`.fbx`を選び、
Inspectorの「インポートスケール」で補正できます
（[エディター](editor.md)の「Model Asset Inspector」を参照）。

## ライティング方針

Scene共通の環境光とDirectional／Point／Spot Lightコンポーネントを実装済みです。

- Inspectorの「シーン環境」で環境光カラーと強度を編集
- GameObjectへ「平行光源」「ポイントライト」「スポットライト」を追加
- Transformの回転から照射方向を計算
- ライト色、強度、範囲、スポット内側／外側角度をInspectorで編集
- 平行光源4灯、ポイントライト8灯、スポットライト4灯まで収集
- ライト設定と環境光を`.scene.json`へ保存
- Scene Viewに方向矢印、範囲球、スポットコーンのギズモを表示
- Directional Lightごとの影、描画距離、濃さ、深度／法線バイアス設定
- 1～4分割のカスケードシャドウと分割バランス設定

新規シーンにはメインカメラと「太陽光」が自動作成されます。既存シーンにも
GameObjectの「コンポーネントを追加」から各ライトを追加できます。環境光は3D描画だけへ
適用され、SpriteRenderer／TextRendererなどの2D描画には影響しません。

MeshRendererは`assets/shaders/TridentLit.hlsl`を使い、ピクセル単位のLambert＋Blinn
ライティング、距離の二次減衰、スポットコーン減衰を計算します。マテリアル上書きを
有効にした互換性のある静的ModelRendererも同じシェーダーを使うため、Point／Spot Lightを
ピクセル単位で計算します。DirectXTKへフォールバックしたモデルでは、各モデル中心で
最も強い方向のライトへ近似します。近似に使える灯数はエフェクトの種類で決まり、
DGSLEffect（CMOのカスタムマテリアルなど）は4灯、BasicEffectやSkinnedEffectなど
それ以外のDirectXTKエフェクトは3灯までです（DirectXTKの仕様上の上限）。

影を有効にした最初のDirectional Lightは、カメラの視錐台を最大4範囲へ分割し、
各範囲を2048×2048のTexture2DArrayへ毎フレーム描画します。近景ほど狭い範囲へ
解像度を集中できるため、単一ShadowMapより遠距離と精細さを両立できます。
Inspectorではカスケード数と、均等分割／近景重視を調整する分割バランスを設定できます。

各正投影範囲はShadowMapのテクセル単位へスナップされ、カメラを少し動かしたときの
影の揺れを抑えます。Litシェーダーではカメラ深度からカスケードを選び、
境界付近10%を次のカスケードとブレンドして切り替え線を抑えます。
影の輪郭は3×3のPCF（比較サンプラーの2×2と合わせて実質4×4相当）で
柔らかくなり、スポット影も3×3、ポイント影は接平面5タップのPCFです。
深度バイアス／法線バイアスでシャドウアクネと浮きを調整できます。
MeshRendererとModelRendererは影を落とします。共通Litシェーダーで描画される
MeshRendererと静的ModelRendererは影を受けます。

2D描画は基本的に非ライティングのまま維持し、必要になった段階で2D Lightを別系統として
追加します。

## Skybox、Fog、Bloom

Inspectorの「シーン環境」から、シーン全体の空・霧・発光表現を設定できます。

- Skybox：上空、地平線、下側の3色と明るさを持つプロシージャル空
- Fog：色、開始距離、終了距離、密度を指定する距離フォグ
- Bloom：しきい値、強度、広がりを指定する全画面ポストエフェクト

Skyboxはカメラの向きから背景を生成するため、外部テクスチャなしで利用できます。
`.dds`キューブマップを指定するとIBL（環境反射・環境拡散）が有効になり、
初回に一度だけGPUでGGX事前畳み込み（粗さ別ミップのスペキュラキューブ）と
コサイン畳み込みの放射照度キューブを生成します。これにより、粗いマテリアルは
ぼけた映り込み、滑らかなマテリアルは鮮明な映り込みと、粗さに正確な環境反射に
なります（BRDF項はLUT不要の解析近似で評価）。
FogはTridentLitシェーダーに加え、DirectXTK標準Effectへフォールバックしたモデルにも
適用されます。BloomはScene View、Game View、エディターなしの通常ゲーム描画に対応し、
明るいライトや加算パーティクルの周囲へ柔らかな発光を加えます。

設定はシーンJSONの`environment.sky`、`environment.fog`、`environment.bloom`へ保存されます。
サンプルシーンでは3機能が有効になっており、Inspectorのチェックボックスで個別に比較できます。

## Litマテリアル

MeshRendererとModelRendererは共通の`LitMaterial`を持ち、Inspectorから次を編集できます。

- ベースカラーとアルファ
- 粗さ
- アルベドテクスチャ
- 法線マップ
- 法線強度

Asset Browserで画像を右クリックし「画像を割り当て」を選ぶと、選択中のMeshRendererでは
アルベドへ、ModelRendererではマテリアル上書きを有効にしてアルベドへ割り当てられます。
Inspectorのアルベド欄／法線マップ欄へ画像を
ドラッグ＆ドロップするか、「選択画像をアルベドへ」「選択画像を法線へ」も使用できます。

マテリアル設定とテクスチャパスはシーンJSONへ保存されます。画像ファイルやフォルダーの
移動・改名では参照が自動更新され、削除時にはSpriteRenderer／ModelRendererと同様に
参照元として警告されます。

Asset Browserのフォルダーまたは空白の右クリックメニューから`.material.json`を
作成できます。MeshRenderer／ModelRendererへドラッグするか、Materialの右クリック
メニューから「Materialを割り当て」で共有Materialを設定します。Inspectorでは次の操作が可能です。

MaterialをAsset Browserで単クリックすると専用Inspectorが開き、ベースカラー、粗さ、
アルベド、法線マップ、Shader、カスタムパラメーターをまとめて編集できます。
「選択オブジェクトへ適用」で、選択中のMeshRenderer／ModelRendererへ直接割り当てられます。
ShaderはMaterial Inspectorおよび各Rendererの候補一覧から選択でき、標準Litへも戻せます。

- 「Materialへ保存」で現在の値を共有ファイルへ保存
- 「再読込」でディスク上の共有値へ戻す
- 「個別化」で現在値を保ったまま共有参照を解除

共有Materialを保存すると、同じシーン内でそのMaterialを参照する全Rendererへ
即座に反映されます。シーンJSONには共有Materialのパスとフォールバック用の現在値が
保存されるため、既存シーンとの互換性も維持されます。

`TridentLit.hlsl`はGeometricPrimitiveのUVを使ってアルベドをサンプリングします。
法線マップ用の接線空間はピクセルシェーダー内の位置／UV微分から生成するため、
プリミティブ頂点へ接線データを追加する必要はありません。法線マップ未設定時は
内部のフラット法線テクスチャを使用します。

### カスタムShader

書き方の解説とサンプルHLSLは[カスタムShaderガイド](shaders.md)に
まとまっています（2Dスプライト用・画面全体ポストエフェクトもこちら）。

Asset Browserのフォルダー内で右クリックし「新規カスタムShader」を選ぶと、
編集可能なHLSL雛形を作成できます。HLSLをダブルクリックしてコードエディターで編集し、
Mesh Renderer／Model RendererのInspectorへドラッグしてMaterialに設定します。
保存したHLSLは実行中も自動再コンパイルされ、失敗時はInspectorへエラーを表示しながら
直前の正常なShader（未成功なら標準Lit）で描画を継続します。

カスタムShaderはShader Model 5.0の`VSMain`と`PSMain`を持ち、雛形と同じ
`ObjectBuffer`レイアウトを使用します。Inspectorの「カスタムShaderパラメーター」から
`CustomParameters[0]`〜`CustomParameters[3]`へ4本の`float4`を渡せます。
Shaderパスと値は`.material.json`（version 2）およびシーンへ保存されます。
静的モデルは位置・法線・UVが共通Lit形式と互換な場合に対応し、スキニングモデルは
従来どおりDirectXTK描画へフォールバックします。

DirectXTK形式のModelRendererは、上書きが無効ならモデル内蔵マテリアルをそのまま使用します。
上書きを有効にした静的モデルが位置・法線・UVを持つ場合はTrident Litへ自動的に切り替わり、
ベースカラー、粗さ、アルベド、法線マップ、Point／Spot Light、Directional Shadowが
MeshRendererと同じ方法で適用されます。Inspectorの「描画経路」で実際の経路を確認できます。

スキニングモデルや法線／UVのないモデルはDirectXTK内蔵Effectへ安全にフォールバックします。
その場合も対応可能な色・粗さ・テクスチャ設定を内蔵Effectへ変換して適用します。
各ModelRendererは独立したEffectを生成するため、同じモデルを複数配置しても設定は漏れません。

## 3D Particle System

GameObjectへ`パーティクルシステム`を追加すると、カメラを向く3Dビルボード粒子を
Scene ViewとGame Viewへ描画できます。Inspectorでは次を編集できます。

- 最大数、毎秒の生成数、寿命、初速、開始サイズ
- 寿命に沿ったサイズ倍率と開始／終了RGBAカラー
- 重力、再生時間、ループ、自動再生、エディターでのリアルタイムプレビュー
- Cone、Sphere、BoxのEmitter形状
- 加算合成／通常アルファ合成と任意のPNGテクスチャ

再生、停止、再開始、32個のBurstをInspectorから試せます。通常アルファ合成では
カメラから遠い順に粒子を描画し、加算合成では発光エフェクト向けの高速な描画を行います。
Scene／Prefab保存、GameObject複製、アセットの名前変更・移動・削除時の参照追従にも対応します。
サンプルSceneの`発光パーティクルサンプル`には、透過済みの
`textures/particle-glow.png`を使った青いエネルギー噴出を配置しています。

## LODと描画カリング

遠距離用の軽量モデルは、それぞれ別の子GameObjectへModelRendererまたは
MeshRendererとして配置し、親へ`LOD Group`を追加します。InspectorでLODごとの
最大距離と表示対象GameObject、完全に非表示にする距離を設定できます。

```cpp
auto& group = scene.CreateGameObject("Building LOD");
auto& high = scene.CreateGameObject("Building High");
auto& low = scene.CreateGameObject("Building Low");
high.SetParent(&group);
low.SetParent(&group);

group.AddComponent<Trident::LODGroupComponent>(
    std::vector<Trident::LODLevel>{
        { 25.0f, high.Id() },
        { 80.0f, low.Id() }
    },
    160.0f);
```

3D描画では次の順番で表示対象を絞り込みます。

1. カメラ距離からLODを1つ選択
2. Frustum Cullingでカメラの視錐台外を除外
3. 32×18の保守的な深度グリッドで、手前の不透明モデルに完全に隠れた物体を除外

OcclusionはGPU Query待ちを発生させない軽量なCPU方式です。Box／Capsule／Sphere Colliderが
ある場合はそのBoundsを使い、それ以外はTransformされた単位Boundsを使います。
正確なカリング範囲が必要な大型モデルにはColliderを追加してください。ワイヤーフレーム
ModelRendererは遮蔽物として扱いません。

Inspectorの「シーン環境 → 描画カリング」で各機能のON/OFFと、総Renderer数、
表示数、LOD／視錐台／遮蔽で除外された数を確認できます。設定はSceneへ保存されます。

## よくあるつまずき

- **画面が暗い／真っ黒** — シーンにライトがあるか確認します（新規シーンには
  「太陽光」が自動で入りますが、削除すると環境光だけになります）。
- **影が出ない** — ライト側の影設定がONか、品質プリセットがLow
  （影無効）になっていないかを確認します。
- **モデルが表示されない** — 対応形式（.gltf/.glb/.fbx/.cmo等）か、
  ModelRendererへ割り当て済みかを確認します。Inspectorの「描画経路」で
  実際にどの経路で描かれているかも確認できます。
- **映り込みがそれっぽくならない** — プロシージャルSkyboxのままでは簡易的な
  環境光のみです。`.dds`キューブマップを設定するとIBL（粗さに正確な反射）が
  有効になります。
- **遠くの物が急に消える** — LOD Groupの非表示距離、またはカリング設定を
  確認します。「シーン環境 → 描画カリング」でどの段階で除外されたかが
  数字で分かります。
- **パーティクルが眩しすぎる／地味すぎる** — 加算合成（発光向け）と
  通常アルファ合成（煙など）の切り替えとBloomのしきい値を調整します。

