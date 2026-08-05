# はじめてのTridentEngine

ゲームエンジンが初めての人向けに、プロジェクト作成から
「キューブを操作して遊べるようにしてエクスポートする」までを
順番に説明します。所要時間はおよそ30分です。

## 0. 準備

- Windows 10/11 と Visual Studio 2022 以降（C++デスクトップ開発）
- リポジトリ直下で次を実行するとエディター一式がビルドされます。

```bash
cmake --preset windows-release
cmake --build --preset windows-release
```

ビルドが終わると`out/build/windows-release`以下に`TridentHub.exe`
（プロジェクトランチャー）と`TridentEditorApp.exe`（エディター）が
できます。

## 1. プロジェクトを作る

1. `TridentHub.exe`を起動します（Unity Hubに相当します）。
2. 「新規プロジェクト」でテンプレートと保存先を選び、
   「作成して開く」を押します。
3. エディターが起動します。

## 2. エディター画面の見かた

| パネル | 役割 |
|---|---|
| ヒエラルキー | シーンにあるGameObjectの一覧（親子関係もここで操作） |
| Scene View | 作業用の3Dビュー。クリックで選択、ギズモで移動/回転/拡縮 |
| Game View | ゲームカメラから見た実際の画面 |
| Inspector | 選択中のGameObjectのコンポーネントを編集 |
| アセット | プロジェクト内のファイル一覧（シーン、モデル、スクリプト等） |
| Console | ログ・警告・エラーの表示 |

## 3. 床とプレイヤーを置く

1. ヒエラルキーを右クリックしてGameObjectを作成し、名前を
   「床」にします。
2. Inspectorの「コンポーネントを追加」から **Mesh Renderer** を
   追加します（形状は立方体のままでOK）。
3. Transformのscaleを `20, 1, 20`、positionを `0, -0.5, 0` に
   します。これで床になります。
4. 物理で受け止められるように **Box Collider 3D** も追加します。
5. 同じ手順でもう1つGameObject「プレイヤー」を作り、
   Mesh Renderer（色はInspectorで好きに変更）、Box Collider 3D、
   **Rigidbody** を追加して、positionを `0, 3, 0` にします。
6. 再生すると、プレイヤーが落下して床の上に乗れば成功です。

ライトが暗いときは、GameObjectへ **Directional Light** を追加して
回転を `-0.8, 0.4, 0` あたりにすると太陽光になります。

## 4. カメラ

GameObject「Main Camera」を作成して **Camera** を追加し、
positionを `0, 4, 10`、回転のピッチ（EulerAnglesのx）を `-0.25` くらいにすると
斜め上からの見下ろしになります。

## 5. C++スクリプトで動かす

1. アセットパネルの余白を右クリック →「新規C++ Script」→
   名前を `PlayerController` にします。
2. 生成されたファイルが開くので、次のように書きます。

```cpp
#include "Trident/Trident.h"

class PlayerController final : public Trident::Script
{
public:
    void Update(const float deltaTime) override
    {
        // プロジェクト設定のInput Action（既定でWASD/ゲームパッド）
        const float moveX =
            Graphics().Input().Value("MoveHorizontal");
        const float moveZ =
            Graphics().Input().Value("MoveVertical");

        // Script基底クラスのショートカットで自分のTransformを操作
        GetTransform().position.x += moveX * 6.0f * deltaTime;
        GetTransform().position.z -= moveZ * 6.0f * deltaTime;
    }
};

TRIDENT_SCRIPT(PlayerController);
```

3. 保存するとバックグラウンドで自動ビルドされ、
   C++ Scriptをヒエラルキーの「プレイヤー」へドラッグ＆ドロップ
   すればアタッチ完了です。
4. 再生してWASDで動けば成功です。

### よく使うショートカットAPI

`Trident::Script`を継承したクラスでは、Unityに近い書き方が
そのまま使えます。

```cpp
auto* body = GetComponent<Trident::RigidbodyComponent>();  // 自分のコンポーネント
auto* gun = GetComponentInChildren<Trident::ParticleSystemComponent>(); // 子孫から
auto* enemy = FindWithTag("Enemy");        // タグで検索
auto* boss = Find("ボス");                  // 名前で検索
auto& bullet = Instantiate("prefabs/bullet.prefab.json"); // Prefab生成
Destroy(Owner());                           // 自分を削除
Invoke(1.5f, [this] { /* 1.5秒後に実行 */ });
StartCoroutine(Intro());  // co_await Trident::WaitForSeconds{1.0f} で待てる関数
```

タグは「ファイル」→「プロジェクト設定...」の「タグ」で登録すると、
InspectorのTag欄にドロップダウンで表示されます。

## 6. 保存とエクスポート

- シーンはメニューから保存します（`assets/scenes/*.scene.json`）。
- 「ファイル」→「プロジェクト設定...」で起動シーン・解像度・
  品質・入力・タグ・ゲームアイコンを設定します。
- 「ファイル」→「ゲームをエクスポート...」で配布フォルダーが
  生成されます。実行ファイルはゲーム名（例: `マイゲーム.exe`）になり、
  設定したアイコンが埋め込まれます。「配布用ZIPも作成」にチェックを
  入れれば、そのまま友達に渡せる.zipまで作られます。
- 動作がおかしいときは、ゲーム中に**F1キー**でデバッグオーバーレイ
  （FPS・オブジェクト数・直近の警告ログ）を表示できます。

## 7. 次のステップ

- サンプルゲーム **TARGET RANGE**
  （`assets/scenes/target-range.scene.json`と
  `samples/GameModule/TargetRangeGame.cpp`）は、タグ・物理・UI・
  影・オーディオなど主要機能をひととおり使った実例です。
- 各機能の詳しい説明は[ドキュメント一覧](index.md)へ。
  物理、NavMesh、UI、アニメーション、オーディオ、Prefabなどが
  日本語でまとまっています。
