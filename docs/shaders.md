# カスタムShader（HLSL）

自分のHLSLで見た目を拡張する方法。3Dマテリアル、2Dスプライト／パーティクル、
画面全体のポストエフェクトの3系統があり、すべて保存するだけのホットリロードに
対応しています。

[← ドキュメント一覧へ戻る](index.md)

**この文書の読み方**

| 知りたいこと | 見る場所 |
|---|---|
| とりあえず動かす | [1分でためす](#1分でためす3dマテリアルshader) |
| 使える変数・テクスチャ枠 | [使える入力](#3dマテリアルshaderで使える入力) |
| **Inspectorに名前付きの調整UIを出す** | [シェーダー宣言](#シェーダー宣言hlslへ書く設定) |
| **半透明・加算・両面描画にする** | [描画状態を指定する](#描画状態を指定する) |
| **マスクなどテクスチャを増やす** | [テクスチャを増やす](#テクスチャを増やす) |
| スプライトやパーティクルに使う | [2DカスタムShader](#2dカスタムshaderスプライトパーティクル) |
| 画面全体を加工する | [ScreenEffect](#画面全体のポストエフェクトscreeneffect) |
| 保存しても反映されない | [ホットリロード](#ホットリロードとエラー表示)／[よくあるつまずき](#よくあるつまずき) |

Shaderを共有Materialとして保存すれば、Prefabやパッケージで配布できます
（マテリアル化の詳細は[グラフィックス](graphics.md)の「Litマテリアル」を
参照してください）。

## 1分でためす（3DマテリアルShader）

1. Asset Browserのフォルダー内で右クリック →「**新規カスタムShader**」で
   名前を入力します。`VSMain`／`PSMain`を持つ編集可能な雛形（Shader Model
   5.0）が作られます。
2. HLSLをダブルクリックしてコードエディターで開き、`PSMain`の返す色を
   少し変えて保存します。**実行中でも保存した瞬間に反映されます。**
   （どのエディターで開くかは「ファイル」→「プロジェクト設定...」→
   「スクリプト」で選べます。）
3. HLSLファイルをMesh Renderer／Model RendererのInspectorへドラッグすると、
   そのMaterialのShaderになります。

InspectorからShaderへ値を渡すには「**カスタムShaderパラメーター**」を
使います。例えば雛形の`PSMain`の最後（returnの直前）に1行足すと、
Inspectorで設定した色を混ぜられます。

```hlsl
// CustomParameters[0]のRGBへ、wの割合で近づける
// （Inspectorのカスタムパラメーター0で色と強さを操作）
color.rgb = lerp(color.rgb,
    CustomParameters[0].rgb,
    CustomParameters[0].w);
```

## 3DマテリアルShaderで使える入力

雛形と同じ`ObjectBuffer`（`b0`）レイアウトを使います。

| 変数 | 内容 |
|---|---|
| `World` / `ViewProjection` / `WorldInverseTranspose` | 変換行列 |
| `MaterialColor` | ベースカラー×アルファ |
| `CameraPosition` / `CameraForward` | カメラのワールド位置と向き |
| `MaterialParameters` | 粗さ等のマテリアル値 |
| `CustomParameters[8]` | Inspectorから渡す自由な`float4` |
| `MaterialTextureParameters` | PBRマップの有効フラグと遮蔽の強さ |
| `EmissiveParameters` | 発光色（`rgb`）と発光マップの有無（`w`） |

`MaterialTextureParameters`と`EmissiveParameters`は`ObjectBuffer`の**末尾**に
あるので、これらの行を持たない既存の自作Shaderもそのまま動きます。

### レジスタの割り当て

どのスロットがエンジン用で、どこが自由に使えるかの一覧です。

| スロット | 用途 |
|---|---|
| `b0` | `ObjectBuffer`（上の表） |
| `b1` | ライティング定数バッファ（`TridentLit.hlsl`が正） |
| `b2` | `BoneBuffer`（スキニング用） |
| `t0` / `t1` | アルベド／法線マップ |
| `t2` | 影マップ（`Texture2DArray`） |
| `t3` | 環境マップ（キューブマップ） |
| `t4` / `t5` | スポット影／ポイント影 |
| `t6` | 放射照度（IBLの拡散） |
| **`t7`〜`t10`** | **自由に使える追加テクスチャ**（宣言で割り当て） |
| `t11` / `t12` | 粗さマップ（`.g`）／金属度マップ（`.b`） |
| `t13` | 遮蔽（AO）マップ（`.r`） |
| `t14` | 発光（emissive）マップ |
| `t15` | 画面空間の遮蔽（SSAO、`.r`）。下の「SSAOを受け取る」を参照 |
| `s0` / `s1` | 標準サンプラー／影比較用サンプラー |

自由枠が`t7`〜`t10`で途切れて`t11`からエンジンに戻るのは、自由枠が
公開済みの取り決めだからです。エンジンの追加分を`t7`以降へ詰めると、
既存の自作Shaderが**別のテクスチャを黙って読む**ことになるため、
番号の連続性より互換性を取っています。

### SSAOを受け取る

SSAO（[グラフィックス](graphics.md#ssao遮蔽による陰り)）は、ライティングの
前に用意された遮蔽テクスチャを**Shader側が読んで環境光へ掛ける**方式です。
`TridentLit.hlsl`をそのまま使うMeshRenderer／ModelRendererには自動で
かかりますが、自作Shaderは次の3行を足すと受け取れます。読まなくても
エラーにはならず、その場合はそのオブジェクトに陰りがかからないだけです。

```hlsl
Texture2D ScreenAmbientOcclusionTexture : register(t15);

// LightingBuffer（b1）の末尾。x=1/画面幅, y=1/画面高さ, z=有効。
// 末尾に足されているので、この行が無い既存Shaderもそのまま動きます。
float4 ScreenAmbientOcclusionParameters;

// ピクセルシェーダーの中。UVはピクセル座標から作ります。
float occlusion = 1.0f;
if (ScreenAmbientOcclusionParameters.z >= 0.5f)
{
    occlusion = ScreenAmbientOcclusionTexture.Sample(
        MaterialSampler,
        input.Position.xy * ScreenAmbientOcclusionParameters.xy).r;
}
```

掛ける先は**環境光やIBLの項だけ**にしてください。直接光へ掛けると影の中が
二重に暗くなり、汚れのように見えます。

## シェーダー宣言（HLSLへ書く設定）

シェーダーのコメント内へ宣言を書くと、エディターとエンジンが読み取って
**パラメーターの名前付け・テクスチャ枠の追加・描画状態の指定**ができます。
HLSLとしてはコメントなので、宣言を消しても動きます。

| 宣言 | 読み取るのは | できること |
|---|---|---|
| `TRIDENT_PROPERTIES` | **エディター**のみ | パラメーターに名前・型・範囲・既定値を付けて、Inspectorへ名前付きUIを出す。テクスチャ枠の追加もここ |
| `TRIDENT_RENDER_STATE` | **エンジン（実行時）** | 半透明・加算合成・カリング・深度書き込みの指定。エクスポートしたゲームでも同じ見た目になる |

### パラメーターに名前を付ける

シェーダーの中へ`TRIDENT_PROPERTIES`宣言を書くと、Inspectorが読み取って
**名前付きの調整UI**を出します（Unityの`Properties`に相当）。数字の意味が
シェーダー自身に書かれるので、Materialとして保存して配っても、受け取った人が
そのまま調整できます。

```hlsl
/* TRIDENT_PROPERTIES
[
  { "target": "0.rgb", "type": "color", "name": "着色",
    "default": [1.0, 1.0, 1.0] },
  { "target": "0.a", "type": "float", "name": "着色の強さ",
    "min": 0.0, "max": 1.0, "default": 0.0 },
  { "target": "2.xy", "type": "vector", "name": "UVの拡大",
    "default": [1.0, 1.0] }
]
*/
```

| 項目 | 内容 |
|---|---|
| `target` | `CustomParameters`の「番号.成分」。成分は`x/y/z/w`でも`r/g/b/a`でも書けます（`"3"`のように省略すると`x`） |
| `type` | `float`（1成分）／`color`（3または4成分のカラーピッカー）／`bool`（0か1）／`vector`（2〜4成分の数値入力）／`texture`（画像の割り当て枠） |
| `name` | Inspectorに出す表示名 |
| `min`／`max` | 両方書くとスライダー、省略すると数値入力 |
| `default` | 「既定値に戻す」で書き込む値。数値、真偽値、配列のいずれか |

- 解釈するのは**エディターだけ**です。シェーダーのコンパイル方法や実行時の
  動作は変わりません（コメント内に書くため、HLSLとしても無視されます）。
- **宣言が無いシェーダーは従来どおり**、生の`float4`を8本編集するUIになります。
  途中から宣言を足しても、消しても動きます。
- シェーダーを保存し直すと宣言も読み直されます（ホットリロードと同じ感覚）。
- 宣言に誤りがあるとInspectorへ理由を表示し、生のfloat4編集へ戻ります。

新規作成した`.hlsl`（雛形）には宣言の実例が入っているので、名前と範囲を
書き換えるところから始められます。

### テクスチャを増やす

エンジンは`t0`（アルベド）、`t1`（法線）、`t2`（影）、`t3`（環境マップ）、
`t4`（スポット影）、`t5`（ポイント影）、`t6`（放射照度）を使います。
**自作Shaderが自由に使える枠は`t7`〜`t10`の4枚**です。マスク、ランプ、
詳細テクスチャなどに使えます。

```hlsl
Texture2D MaskTexture : register(t7);

/* TRIDENT_PROPERTIES
[
  { "target": "t7", "type": "texture", "name": "マスク" }
]
*/
```

宣言するとInspectorに割り当て枠が出て、画像をドロップするか
「選択中の画像を割り当て」で設定できます。**未設定の枠は白テクスチャ**が
渡るので、シェーダー側で「設定されているか」を分岐する必要はありません
（掛け算するだけで無効時は素通しになります）。

割り当てはMaterialにもRendererにも保存され、画像を移動・改名しても
GUIDで参照が追従します。

ライト計算まで自分で行いたい場合は、エンジンのライティング定数バッファ
（`b1`）を`TridentLit.hlsl`からコピーして使います。その際は
**配列サイズや末尾の`float4 ShadowTexelSizes;`まで正確に一致**させて
ください（エンジン更新でレイアウトは末尾へ伸びることがあり、
`TridentLit.hlsl`が常に最新の正です）。

スキニング（ボーン）モデルにもカスタムShaderを使えます。雛形の
`VSSkinnedMain`がボーン変形（`BoneBuffer`、最大72ボーン）を行うので、
キャラクターにも自作Shaderが当たります。エンジンはスキニング用に
別途コンパイルし、専用の入力レイアウトを作ります。`VSSkinnedMain`を
消すとそのモデルはDirectXTK描画へフォールバックします。

Shaderを指定していないモデルは標準の`TridentLit.hlsl`で描かれます
（[グラフィックス](graphics.md)の「モデルのマテリアル取り込み」参照）。

スキニング時のピクセルシェーダーは`PSMain`ではなく`PSSkinnedMain`が
呼ばれ、入力は`SkinnedPixelInput`です。頂点変形はDirectXTKの
`SkinnedEffect`が担当し、エンジンはピクセルシェーダーだけを差し替える
ため、`PixelInput`とはメンバーの並び（セマンティクス）が違います。
雛形のとおり詰め替えてから共通の陰影関数へ渡してください。

### 描画状態を指定する

半透明や加算合成、両面描画、深度書き込みの有無を**Shader側から**指定できます。
こちらは実行時にも使われる情報なので、ゲームでも同じ見た目になります。

```hlsl
/* TRIDENT_RENDER_STATE
{ "blend": "additive", "cull": "none", "depthWrite": false }
*/
```

| 項目 | 値 |
|---|---|
| `blend` | `opaque`（既定）／`alpha`（ガラス・フェード）／`additive`（発光・エフェクト）／`premultiplied` |
| `cull` | `back`（既定・裏面を描かない）／`front`（内側から見せる箱）／`none`（板ポリゴンの草や旗） |
| `depthWrite` | 深度バッファへ書き込むか。**`blend`が`opaque`以外で省略すると自動で`false`** になります（半透明の重ね合わせで抜けを防ぐため） |
| `depthTest` | 深度テストをするか。`false`で常に手前へ描きます |

宣言が無いShaderは従来どおり（不透明・裏面カリング・深度書き込みあり、
ベースカラーのアルファが1未満なら半透明扱い）です。

**GPUインスタンシングでも効きます。** 同じ形状・同じShader・同じテクスチャの
MeshRendererは自動で1回のドローコールにまとめられますが（バッチのキーに
Shaderが含まれるため、まとめられた分の宣言は必ず同じです）、その場合も
宣言した合成・カリング・深度が適用されます。加算合成のエフェクトを大量に
並べても1ドローコールで描けます。

ただし**3Dの半透明は前後関係でソートされません**（インスタンシングの有無に
かかわらず）。重なりが気になる場合は、`additive`のように順序に依存しない
合成を使うか、`depthWrite`を切って重ね合わせの破綻を避けてください。

**スキニング（ボーン）モデルでも効きます。** ワイヤーフレーム表示中だけは
デバッグ優先で宣言を無視します。

## 2DカスタムShader（スプライト／パーティクル）

Sprite RendererとParticle SystemはInspectorへHLSLをドラッグすると、
**ピクセルシェーダーだけ**を差し替えられます（頂点処理はそのまま、
エントリポイントは`PSMain`のみ）。8本の`float4`パラメーターと
エラー表示、ホットリロードに対応します。

そのまま使える最小の例（Inspectorのパラメーター0のxを上げると
白くフラッシュ）:

```hlsl
Texture2D SpriteTexture : register(t0);
SamplerState SpriteSampler : register(s0);

cbuffer SpriteParameters : register(b0)
{
    float4 CustomParameters[8];
};

float4 PSMain(
    float4 position : SV_Position,
    float4 color : COLOR0,
    float2 uv : TEXCOORD0) : SV_Target
{
    float4 pixel =
        SpriteTexture.Sample(SpriteSampler, uv) * color;
    pixel.rgb = lerp(
        pixel.rgb,
        float3(1.0, 1.0, 1.0),
        CustomParameters[0].x);
    return pixel;
}
```

- スプライトでは`CustomParameters[5]`（tint）、`[6]`（描画矩形: left,
  top, width, height）、`[7]`（UIビューポート: width, height,
  テクスチャwidth, height）は**エンジンが毎描画設定**します。
  自由に使えるのは`[0]`～`[4]`です。
- Particle SystemはInspectorで**補助テクスチャ**を1枚割り当てられ、
  `t1`から読めます（未設定時は白）。ノイズやマスクに便利です。
- Shaderパスとパラメーターはシーンへ保存され、アセットの改名・移動にも
  参照が追従します。

## ノイズ関数

`assets/shaders/TridentNoise.hlsli` を `#include` すると、5種類のノイズが
使えます。**C++側の `Trident::Noise`（`Trident/Core/Noise.h`）と同じ値を
返す**ので、C++で地形メッシュを作り、シェーダーで同じノイズから草の分布や
色を決める、という組み合わせが成立します。

```hlsl
#include "TridentNoise.hlsli"
float height = TridentFractalNoise2D(uv * 8.0f, 5, 2.0f, 0.5f);
```

```cpp
const float height =
    Trident::Noise::FractalValue2D(x * 0.05f, z * 0.05f, 5);
```

| 関数 | 見た目 | 向いている用途 |
|---|---|---|
| `TridentValueNoise2D` / 1D / 3D | 滑らかな斑 | 基本。軽い揺らぎ |
| `TridentPerlinNoise2D` | うねる有機的な帯 | 岩肌、水面、風 |
| `TridentFractalNoise2D` | 雲 | 地形の高さ、雲、煙 |
| `TridentWorleyNoise2D` | セル状の粒 | 石畳、ひび割れ、泡、鱗 |
| `TridentCurlNoise2D` | 渦（発散のない流れ） | 煙・水のパーティクル |

**乱数との違い**: 乱数は隣の座標でも値が飛びますが、ノイズは近い座標なら
近い値を返します。この連続性が「自然なムラ」を作ります。

`assets/shaders/TridentNoiseSample.hlsl` が見本です。マテリアルへ割り当てると
Inspectorから種類・大きさ・重ねる回数・流れる速さを触れます。

**実装で気をつけていること**（改造するとき用）: ハッシュは整数演算です。
`frac(sin(dot(...)))` の定番手法は環境によって精度が違い、CPUと一致しません。
0〜1への変換は上位24ビット÷16777216（floatの仮数に収める）、補間は5次の
スムーズステップ（3次だと格子線が見える）。片方を直したら必ず両方直して
ください（テストが代表点での一致を検査します）。

## レトロ3D（PS1風）— 同梱の見本

`assets/shaders/TridentRetro3D.hlsl` を3Dオブジェクトのマテリアルへ
割り当てると、当時のハードウェアが「できなかったこと」を再現できます。
Inspectorから9項目を調整できます。

| 項目 | 何が起きるか |
|---|---|
| テクスチャの泳ぎ | 0で現代の正しい遠近、1で当時どおり |
| 頂点のカクつき／格子数 | 頂点を格子へ丸める。数値が小さいほど粗い |
| 色の段数／ディザの強さ | 色数を落として網目状に混色 |
| テクスチャを補間しない | ドット感（ポイントサンプリング） |

肝は1行です。UVを渡す変数へ `noperspective` を付けると、GPUが遠近
補正をやめて画面空間の線形補間になります。これがPS1の挙動そのものです。

```hlsl
noperspective float2 AffineTexCoord : TEXCOORD2;
```

**使うときのコツ**

- **テクスチャを貼ってください。** 歪むのはUVなので、模様が無いと
  泳ぎは目に見えません（無地だと色の量子化しか分かりません）
- **床は格子状に分割してください。** Plane1枚の巨大な床を浅い角度で
  見ると遠近が完全に失われ、模様が横帯に伸びて「壊れて見える」状態に
  なります。小さいPlaneを並べると、当時らしい「揺れ」になります
  （当時のゲームも分割で破綻を抑えていました）
- 解像度自体を落としたいときは、品質設定の「レンダースケール」を
  下げてください（このShaderとは独立に効きます）

**制約**: スキニングモデル（人物など）には泳ぎが出ません。DirectXTKの
頂点シェーダー出力の並びに合わせる必要があり、補間指定を変えられない
ためです（色の量子化と陰影は効きます）。静的なモデル・地形・建物は
完全に効きます。

当時の「ポリゴンの前後がおかしくなる」現象は再現しません。深度バッファが
無くポリゴン単位で並べ替えていたことに由来し、正確に真似ると味ではなく
不具合に見えるためです。

## 画面全体のポストエフェクト（ScreenEffect）

画面全体を入力テクスチャとして受け取る任意のHLSLを、3D合成
（Bloom／トーンマッピングの後、FXAAの前）へ差し込めます。
C++から**毎フレーム**`QueueScreenEffect`を呼びます（登録はその
フレーム限りなので、Updateで呼び続けます）。

```cpp
void Update(float deltaTime) override
{
    m_time += deltaTime;
    Trident::ScreenEffectRequest effect;
    effect.shader = "shaders/sepia.hlsl"; // assets内の相対パス
    effect.customParameters[0] = { 0.8f, 0.0f, 0.0f, 0.0f };
    std::string error;
    if (!Graphics().QueueScreenEffect(effect, nullptr, &error))
    {
        // errorにコンパイルエラー等が入ります
    }
}
```

HLSL側は`VSMain`と`PSMain`を持ち、フルスクリーン三角形として
実行されます。`t0`が描画済みの画面、`t1`／`t2`が
`ScreenEffectRequest::auxiliaryTextures`（未設定時は白）です。

```hlsl
cbuffer ScreenParameters : register(b0)
{
    float4 CustomParameters[8];
    float4 ScreenSize; // xy = 画面の幅・高さ
};

Texture2D SceneTexture : register(t0);
SamplerState SceneSampler : register(s0);

struct VertexOutput
{
    float4 Position : SV_Position;
    float2 TexCoord : TEXCOORD0;
};

VertexOutput VSMain(uint vertexId : SV_VertexID)
{
    // 3頂点で画面全体を覆う定番のフルスクリーン三角形
    VertexOutput output;
    float2 uv = float2((vertexId << 1) & 2, vertexId & 2);
    output.Position =
        float4(uv * float2(2, -2) + float2(-1, 1), 0, 1);
    output.TexCoord = uv;
    return output;
}

float4 PSMain(VertexOutput input) : SV_Target
{
    float4 color =
        SceneTexture.Sample(SceneSampler, input.TexCoord);
    // CustomParameters[0].xの割合でセピア化する例
    float3 sepia = float3(
        dot(color.rgb, float3(0.393, 0.769, 0.189)),
        dot(color.rgb, float3(0.349, 0.686, 0.168)),
        dot(color.rgb, float3(0.272, 0.534, 0.131)));
    color.rgb = lerp(color.rgb, sepia, CustomParameters[0].x);
    return color;
}
```

複数回Queueすると登録順に連結され、前の結果が次の入力になります。
2D／UIはポストエフェクトの**後**に合成されるため、ScreenEffectの
影響を受けません（UIの色はそのまま保たれます）。

## ホットリロードとエラー表示

3系統とも、HLSLを保存すると実行中でも自動で再コンパイルされます。
コンパイルに失敗したときは、Inspector（ScreenEffectは`error`引数）へ
エラーを表示しながら、**直前に成功したShader**（一度も成功していない
場合は標準Shader）で描画を続けます。ゲームが止まることはありません。

## よくあるつまずき

- **編集しても見た目が変わらない** — コンパイルエラーで直前のShaderに
  フォールバック中かもしれません。InspectorのShaderエラー欄を確認して
  ください。
- **エントリポイント名を変えたら動かない** — `VSMain`／`PSMain`は
  固定名です（2Dスプライト／パーティクルは`PSMain`のみ）。
- **b1をコピーしたら影や色が壊れた** — ライティング定数バッファの
  レイアウト不一致です。`TridentLit.hlsl`から丸ごとコピーし直すのが
  確実です（エンジン更新後は特に）。
- **スキニングモデルに効かない** — 雛形から`VSSkinnedMain`または
  `PSSkinnedMain`を消していると、DirectXTK描画へフォールバックします。
  ModelRendererの「従来のDirectXTK描画を使う」がオンになっていないかも
  確認してください（Inspectorの「描画経路」で実際の経路が分かります）。
- **2Dでパラメーター[5]～[7]に書いた値が消える** — その3本はエンジンが
  毎描画上書きします。自由用途には`[0]`～`[4]`を使ってください。
- **ScreenEffectが1フレームで消える** — Queueはそのフレーム限りです。
  効果を出し続ける間は`Update`で毎フレーム呼びます。
