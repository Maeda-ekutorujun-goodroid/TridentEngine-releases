# カスタムShader（HLSL）

自分のHLSLで見た目を拡張する方法。3Dマテリアル、2Dスプライト／パーティクル、
画面全体のポストエフェクトの3系統があり、すべて保存するだけのホットリロードに
対応しています。

[← ドキュメント一覧へ戻る](index.md)

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

テクスチャは`t0`（アルベド）、`t1`（法線マップ）、影マップは
`t2`（`Texture2DArray`）、サンプラーは`s0`／`s1`（影比較用）です。

ライト計算まで自分で行いたい場合は、トゥーン描画の実例
`TridentToon.hlsl`を出発点にするのが近道です（エンジン本体の
`assets/shaders/`に入っています。配布ZIPにも同梱されているので、
プロジェクトへコピーして使ってください）。
エンジンのライティング定数バッファ（`b1`）をコピーして使う場合は、
**配列サイズや末尾の`float4 ShadowTexelSizes;`まで正確に一致**させて
ください（エンジン更新でレイアウトは末尾へ伸びることがあり、
TridentToon.hlslが常に最新の正です）。

スキニング（ボーン）モデルはカスタムShaderに対応せず、従来どおり
DirectXTKの描画へ安全にフォールバックします。

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
  レイアウト不一致です。`TridentToon.hlsl`から丸ごとコピーし直すのが
  確実です（エンジン更新後は特に）。
- **スキニングモデルに効かない** — 仕様です。スキン付きモデルは
  DirectXTK描画へフォールバックします。
- **2Dでパラメーター[5]～[7]に書いた値が消える** — その3本はエンジンが
  毎描画上書きします。自由用途には`[0]`～`[4]`を使ってください。
- **ScreenEffectが1フレームで消える** — Queueはそのフレーム限りです。
  効果を出し続ける間は`Update`で毎フレーム呼びます。
