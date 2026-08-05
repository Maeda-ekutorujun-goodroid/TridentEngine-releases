# UIと2D機能

UI Canvas / Button / 各ウィジェット、2D Tilemap、ゲーム内日本語テキスト。

[← ドキュメント一覧へ戻る](index.md)

## 1分でためす（押すと反応するボタン）

1. GameObjectへ「**UI Canvas**」を追加します（画面全体の基準になります）。
2. その**子**GameObjectへ「**UI Rect Transform**」と「**UI Button**」を追加し、
   ラベルに「スタート」などを入力します。
3. ButtonのInspectorの「**クリック時のイベント**」へ`StartGame`と入力します。
4. 反応させたいGameObjectのスクリプトで、イベント名を購読します。

```cpp
#include "Trident/Trident.h"

class TitleScreen final : public Trident::Script
{
public:
    void Start() override
    {
        // ボタン側にコードは不要。イベント名だけでつながります
        On("StartGame", [this]
        {
            GetScene().Scenes().RequestLoadAsync(
                "scenes/stage-01.scene.json");
        });
    }
};

TRIDENT_SCRIPT(TitleScreen);
```

購読はスクリプトの破棄時に自動解除されます。ボタンを介さず
スクリプト同士で通知したい場合も同じ仕組みで`Emit("イベント名")`が使えます。

## UI CanvasとButton

`UI Canvas`は基準解像度と「幅／高さのどちらへ合わせるか」を持ち、子GameObjectの
`UI Rect Transform`を実際のGame View解像度へ変換します。Rect Transformでは
Anchor Min／Max、Pivot、Anchored Position、Size Deltaを編集でき、中央、四隅、
全画面Stretchのプリセットも選択できます。

UI Rect Transformを持つGameObjectへSpriteRendererまたはTextRendererを追加すると、
通常のTransform座標ではなく計算済みUI領域へ配置されます。`UI Button`は背景色または
任意の画像、日本語ラベル、通常／Hover／Pressed／Disabled色を持ちます。エディターの
Game View内マウス座標と、エクスポートしたゲームのクライアント座標の両方で動作します。

C++からクリックを処理する場合:

```cpp
if (auto* button = object.GetComponent<Trident::UIButtonComponent>();
    button != nullptr && button->ConsumeClick())
{
    // シーン切り替えやゲーム開始処理
}
```

サンプルSceneの右下には解像度変更へ追従する`クリックできます`ボタンを配置しています。
PlayしてGameタブを開くと、マウス操作に応じた色の変化を確認できます。

ButtonのInspectorには「クリック時のイベント」欄があり、イベント名（例:
`StartGame`）を設定すると、クリック時にSceneのイベントバスへ発行されます。
C++ Scriptは`On("StartGame", [this]{ ... });`と書くだけで反応でき、
ボタン側にコードは不要です（詳細は[C++スクリプティング](scripting.md)の
イベント節へ）。

## スプライトアニメーション（Sprite Animator）

Sprite Rendererと同じGameObjectへ`Sprite Animator`を追加すると、
1枚のスプライトシートをコマ送りするフリップブックアニメーションを再生できます。

1. Sprite Rendererへ歩行アニメ等を並べたスプライトシート画像を割り当てます。
2. Sprite Animatorの「シート分割（列×行）」でシートのコマ数を指定します
   （コマ番号は左上から右へ、次の行へと数えます）。
3. 「クリップを追加」で`walk`のようなアニメーションを定義し、
   開始コマ・コマ数・コマ/秒・ループを設定します。
4. 既定クリップは再生開始時に自動で流れます（「自動再生」で無効化できます）。

スクリプトからの切り替えは`Play`を使います。

```cpp
auto* animator = GetComponent<Trident::SpriteAnimatorComponent>();
animator->Play("jump");            // ループしないクリップは最終コマで停止
if (!animator->IsPlaying())
{
    animator->Play("idle");
}
```

スプライトの見た目そのものをHLSLで加工したい場合（フラッシュ、
ディゾルブ等）は[カスタムShaderガイド](shaders.md)の2D節を参照して
ください。

内部的にはSprite Rendererの`SetSourceRect`（テクスチャの正規化部分矩形）を
毎フレーム更新しています。`SetSourceRect`は単体でも使えるため、
アトラスから1コマだけを表示する静的スプライトにも利用できます。
シート分割・クリップ・既定クリップはシーンJSONへ保存され、複製やPrefabにも
引き継がれます。

## 2Dライティング（Light2D）

`Light2D`コンポーネントを追加したGameObjectは、Transformの位置を中心に
半径・色・強度を持つ光源になります。ワールド空間の`Sprite Renderer`
（`UI Rect Transform`を持たないもの）を加算式に照らします。

```cpp
auto& torchLight = torch.AddComponent<Trident::Light2DComponent>();
torchLight.SetColor({ 1.0f, 0.7f, 0.3f });
torchLight.SetIntensity(1.5f);
torchLight.SetRadius(200.0f);
```

- **暗くはなりません。** 光源から遠いスプライトは通常どおりの明るさの
  ままで、近いスプライトへ色と明るさが加算されます。画面全体を暗くする
  「昼夜の陰影」のような用途ではなく、ランタン・ネオン・魔法陣のような
  「光る」演出向けです。
- 対象は独自Shaderを持たない`Sprite Renderer`のみです。UI、Tilemap、
  Particle System、独自HLSLを設定済みのSpriteは対象外です（独自Shader側で
  同様の効果が欲しい場合は[カスタムShaderガイド](shaders.md)を参照して
  ください）。
- 1つのスプライトに影響するのは最も近い2灯までです。
- Radius・色・位置（画面ピクセル基準）はこのエンジンのワールド座標系
  そのまま（1ワールド単位＝1ピクセル）を使うため、Sprite RendererのSize
  と同じ感覚で数値を決められます。
- **Source Rect（アトラス部分表示）と併用すると全体表示になります。**
  内部的にはSV_Positionから作り直したUVでテクスチャをサンプルしており
  （このSpriteBatchはカスタムPS切替時にCOLOR0／TEXCOORD0をそのまま
  信頼できないための対策）、Source Rectの矩形情報までは復元していない
  ためです。スプライトシートの1コマだけを照らしたい場合は現状非対応です。

## Sprite Mask（表示範囲の切り抜き）

`Sprite Mask`コンポーネントを追加したGameObjectは、Transformの位置を
中心に矩形または円の範囲を持つマスクになります。`Sprite Renderer`の
「Sprite Mask」欄で`マスクの内側だけ表示`／`マスクの外側だけ表示`を
選ぶと、そのSpriteが最も近いマスクでクリップされます。フォグ・オブ・
ウォーの視界、懐中電灯の明かり、ウィンドウ状の演出に使えます。

```cpp
auto& mask = maskObject.AddComponent<Trident::SpriteMaskComponent>();
mask.SetShape(Trident::SpriteMaskShape::Circle);
mask.SetSize({ 300.0f, 300.0f }); // 円は幅の値を直径として使います

auto& fogSprite = fog.AddComponent<Trident::SpriteRendererComponent>();
fogSprite.SetMaskInteraction(
    Trident::SpriteMaskInteraction::VisibleOutsideMask);
```

- クリップは境界がくっきりした二値判定です（半透明フェードなし）。
- マスク形状は矩形・円のみです。任意画像の透明部分を型として使う
  Unity風のアルファマスクには対応していません。
- 対象は独自Shaderを持たない`Sprite Renderer`のみで、1つのSpriteが
  従うのは最も近いマスク1つだけです。Light2Dと同時に必要なSpriteでは
  Sprite Maskが優先されます（バッチ切替で使えるカスタムShaderは
  1つのため、同時使用はできません）。
- Light2Dと同じくSource Rectの矩形情報は復元していないため、
  アトラス使用時は全体表示になります。

## 2D TilemapとTile Palette

GameObjectの「コンポーネントを追加」から`Tilemap`を追加できます。Tilemapは1枚の
タイルシートを列数・行数で分割し、各グリッドセルにはタイル番号だけを保持します。
負座標を含む任意の位置へ配置でき、セルサイズ、Atlas分割、全体色とアルファを設定できます。
配置セルとタイルシート参照はScene／PrefabのJSONへ保存されます。Inspectorの
「描画順」（Sprite RendererのSortOrderと同じ尺度、数値が大きいほど手前）を
使うと、背景／地形／前景のように複数のTilemapを重ねたときの表示順を制御できます。

「タイルパレット」タブは他のEditorタブと同様に、タブをドラッグしてDock位置の変更、
独立ウィンドウ化、サイズ変更ができます。使い方は次の通りです。

1. HierarchyでTilemapを持つGameObjectを選択
2. Asset Browserからタイルシート画像をパレットへドロップ
3. 画像の列数・行数とゲーム上のセルサイズを設定
4. パレットでタイルを選択し、Sceneタブを左ドラッグして配置
5. 「消去」へ切り替えて左ドラッグするとセルを削除

Sceneタブにマウスがある間は`B`でペイント、`E`で消去へ切り替えられます。一筆分の
連続編集が1回のUndo履歴になるため、`Ctrl+Z`でまとめて取り消せます。タイルシートの
ファイル移動・改名・削除確認はAsset Browserの参照管理にも統合されています。

### 当たり判定の自動生成

InspectorのTilemapコンポーネントにある「コライダーを生成」ボタンを押すと、配置済みの
セルをすべて衝突対象として、隣接セルをまとめた最小限の`BoxCollider2D`を子GameObject
として自動生成します（貪欲な矩形マージ。タイル1枚ごとにColliderを置くより描画・判定
コストが小さく済みます）。生成されたColliderはTilemapの子として`タイルコライダー（自動
生成）`という名前で並び、通常のBox Collider 2Dと同じくSceneのJSONへ保存されます。
セルを編集したら再度「コライダーを生成」を押してください（既存の生成済みColliderは
置き換えられます）。現状は配置済みセル全体が衝突対象になるため、当たり判定の要らない
背景・装飾タイルは衝突を持たせたいTilemapとは別のTilemap GameObjectへ分けてください。

### 奥行きのあるスクロール（Parallax Layer）

`Parallax Layer`コンポーネントを背景・前景のTilemapやSpriteへ追加すると、
参照（既定はMain Camera）の移動量に倍率を掛けた分だけ自身を動かします。

```cpp
auto& parallax =
    background.AddComponent<Trident::ParallaxLayerComponent>();
parallax.SetFactor({ 0.3f, 0.3f }); // カメラの30%の速さで動く遠景
```

- 倍率1.0で参照と同じ速さ（奥行きなし）、0.5で半分の速さ（遠い背景）、
  0で画面に固定（空のような最遠景）、1.0より大きいと参照より速く動きます
  （近景）。X／Yを別々に設定できます。
- 参照はInspectorの「参照」欄でGameObjectを直接指定することもできます
  （未設定＝Main Camera追従が既定です）。
- 初回更新時点の位置を原点として記録するため、Playを開始した瞬間から
  参照が動いた分だけ追従します。参照を実行中に切り替えると、その時点の
  位置を新しい原点として記録し直します。

## ゲーム内日本語テキスト

TextRendererはDirectWriteでWindowsフォントを透過テクスチャへ変換し、
SpriteBatchでGame Viewへ描画します。

Inspectorでは次を編集できます。

- UTF-8テキスト
- 文字サイズ
- 文字色
- フォントファミリー
- レイアウト幅／高さ（0は自動サイズ）
- 自動折り返し
- 左／中央／右揃え
- 上／中央／下揃え

固定レイアウト枠を指定すると、その範囲内で整列と折り返しが行われます。
日本語文字テクスチャは、テキスト・書式・レイアウト設定ごとにAssetManagerで
キャッシュされます。

## よくあるつまずき

- **ボタンが押せない** — ButtonのGameObjectが**UI Canvasの子**で、
  **UI Rect Transform**を持っているかを確認します。
- **UIの位置が解像度でずれる** — Rect TransformのAnchorを画面の四隅や
  中央へ正しく設定します（右下に置くUIは右下Anchorに）。
  Canvasの基準解像度と「幅／高さのどちらへ合わせるか」も確認します。
- **スプライトアニメが動かない** — Sprite Animatorの「シート分割（列×行）」を
  設定したか、`Play("名前")`のクリップ名が定義と一致しているかを確認します。
- **ループしないアニメの終わりを検出したい** — `IsPlaying()`がfalseに
  なったタイミングで次のクリップへ切り替えます（ページ内サンプル参照）。
- **文字が表示されない** — TextRendererのフォントファミリー名がWindowsに
  存在するか、文字色のアルファが0になっていないかを確認します。
- **UIが3Dの後ろに隠れる／色が変わる** — UIと2Dはポストエフェクトの後に
  合成されるため通常は最前面・原色のままです。SpriteRendererをUI Rect
  Transformなしで使うとワールド空間の2Dとして扱われる点に注意してください。

