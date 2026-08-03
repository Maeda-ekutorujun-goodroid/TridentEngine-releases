# C++スクリプティング（Game Module）

C++ Scriptの作成、ホットリロード、Script APIとコンポーネントの例。

[← ドキュメント一覧へ戻る](index.md)

## ランタイムとエディター

ビルド時に次の独立した成果物が生成されます。

- `TridentRuntime.dll`: ゲームループ、シーン、描画、物理、アセット
- `TridentEditor.lib`: Dear ImGui／ImGuizmoを使う編集機能
- `TridentEditor.exe`: 任意のTridentプロジェクトを編集するシーンエディター
- `TridentHub.exe`: 新規作成、既存プロジェクト、最近使ったプロジェクトのランチャー
- `TridentGame.exe`: Runtimeだけを使うエディターなしゲーム
- `TridentGameModule.dll`: ゲーム固有C++ Component

`TridentGame.exe`は`TridentRuntime.dll`へ動的リンクしますが、Dear ImGui、
ImGuizmo、ファイルダイアログなどのエディター機能には依存しません。

## C++ Game Module

ゲーム固有コードは`TridentRuntime`本体を変更せず、`TridentGameModule.dll`へ分離できます。
DLLは起動時に自動検出され、登録された型はInspectorの
「コンポーネントを追加」→「Game Module (C++)」に表示されます。
Unityに近い作成手順をエディターから利用できます。

1. アセットウィンドウの任意のフォルダーまたは余白を右クリックします。
2. 「新規C++ Script」を選び、`PlayerController.cpp`のような名前を入力します。
3. 生成されたファイルがコードエディターで開きます。
4. `Update`、60 Hzの`FixedUpdate`、`OnCollisionEnter`などへ処理を書きます。
5. C++ ScriptをヒエラルキーのGameObject、またはInspector下部の
   「コンポーネントを追加」へドロップします。
6. バックグラウンドで自動ビルド・Hot Reloadされ、成功するとそのGameObjectへ
   Scriptが自動でアタッチされます。

手動で追加する場合は、アセットの右クリックメニューから「Game Moduleをビルド」を選び、
Inspectorの「コンポーネントを追加」→「Game Module (C++)」から型を選択できます。

`.cpp`はAsset Browserでダブルクリックしても開けます。どのエディターで開くかは
「ファイル」→「プロジェクト設定...」→「スクリプト」で選択します（Visual Studio Code
とVisual Studioは自動検出、未設定ならWindowsのファイル関連付け。`.hlsl`も同じ設定
です）。詳しくは[プロジェクト](project.md)の「プロジェクト設定」を参照してください。

使える機能の一覧は[コード一覧](code-reference.md)にまとまっています
（宣言・引数・戻り値・サンプル付き）。困ったときの逆引き表もあります。

生成コードは`Trident::Script`を継承し、必要な関数だけを上書きする初心者向けAPIです。

```cpp
#include "Trident/Trident.h"

class PlayerController final : public Trident::Script
{
public:
    void Update(float deltaTime) override
    {
        // 毎フレームの処理
    }
};

TRIDENT_SCRIPT(PlayerController);
```

`Start`、`Update`、`FixedUpdate`、`OnCollisionEnter/Stay/Exit`のうち、
使うものだけを記述します。所有GameObjectは`Owner()`、GraphicsDeviceは`Graphics()`から
取得できます。Create／Destroy、関数ポインター、ホットリロード用の保存、
Game Module登録はエンジン側が自動生成します。プロジェクト専用CMakeは`assets`以下の
`.cpp`を自動検出するため、`CMakeLists.txt`や登録一覧の編集は不要です。

Script基底クラスにはUnityのMonoBehaviourに相当するショートカットが
あり、`Owner()`や`GetScene()`を書かずに主要な操作を直接呼べます。

```cpp
// 自分のGameObjectを操作
GetTransform().position.y += 1.0f;
auto* body = GetComponent<Trident::RigidbodyComponent>();
auto& audio = AddComponent<Trident::AudioSourceComponent>();

// 自分の階層から探す（Unityと同じく自分自身も含みます）
auto* muzzle =
    GetComponentInChildren<Trident::ParticleSystemComponent>();
auto* root = GetComponentInParent<Trident::RigidbodyComponent>();
auto* weapon = Owner().FindChild("腕/手/武器"); // パス指定

// シーン全体から探す
auto* player = FindWithTag("Player");
auto enemies = FindObjectsWithTag("Enemy");
auto* boss = Find("ボス");
auto* camera =
    GetScene().FindComponentOfType<Trident::CameraComponent>();

// 生成と削除
auto& bullet = Instantiate("prefabs/bullet.prefab.json");
Destroy(*enemyObject);
```

`GetComponentInChildren`系は既定で非アクティブ階層をスキップし、
引数に`true`を渡すと含めます。

### タイマーとコルーチン

一定時間後の処理は`Invoke`（1回）／`InvokeRepeating`（繰り返し）で
予約できます。「待って→実行して→また待つ」のような流れは、
コルーチンで上から順に書けます（Unityの`yield return`に相当）。

```cpp
class BossIntro final : public Trident::Script
{
public:
    void Start() override
    {
        StartCoroutine(Intro());
    }

    Trident::Coroutine Intro()
    {
        SetBossVisible(true);
        co_await Trident::WaitForSeconds{ 2.0f };   // 2秒待つ
        PlayRoar();
        co_await Trident::WaitUntil{
            [this] { return m_playerInRange; } };   // 条件成立まで待つ
        co_await Trident::WaitForNextFrame{};       // 1フレーム待つ
        StartBattle();
    }
};
```

- 待機はゲーム時間（`Time::SetTimeScale`適用後）で進みます。
- `StartCoroutine`はハンドルを返し、`StopCoroutine(handle)`／
  `StopAllCoroutines()`で停止できます。Scriptの破棄時には
  自動で全コルーチンが停止します。
- 待機条件は`WaitForSeconds`／`WaitForNextFrame`／`WaitUntil`／
  `WaitWhile`の4種類です。
- コルーチン内で自分のGameObjectを`Destroy`する場合は、
  それをコルーチンの最後の文にしてください。

### イベント（名前で連携するシグナル）

「敵が倒された」「ゲーム開始」のような出来事を名前で購読・発行できます
（Godotのシグナルに相当）。受け取る側と発行する側が互いを参照しないため、
UIとゲームロジックの連携が疎結合になります。

```cpp
// 受け取る側（スコア管理Script）
void Start() override
{
    On("EnemyDied",
        [this](const Trident::EventArgs& args)
        {
            m_score += static_cast<int>(args.number);
        });
    On("StartGame", [this] { BeginRound(); });  // 引数なしでもOK
}

// 発行する側（敵Script）
void OnDestroy() override
{
    Trident::EventArgs args;
    args.number = 100.0f;  // スコア
    Emit("EnemyDied", args);
}
```

- `EventArgs`は`sender`（発行元GameObject）、`number`、`text`を持つ
  固定の入れ物です。`Emit`では`sender`が自動で自分になります。
- 購読はScriptの破棄時に自動解除されます（`Off(handle)`で手動解除も可）。
- イベントバスはSceneの`Events()`が実体で、Scene切り替えでは消えません
  （`DontDestroyOnLoad`の常駐Scriptの購読が維持されます）。
- **UI Buttonの「クリック時のイベント」**にイベント名を設定すると、
  ボタン側はコード不要で、Script側の`On("イベント名", ...)`だけで
  クリックへ反応できます。

ビルド結果はプロジェクトごとの`.trident/bin/TridentGameModule.dll`へ出力され、
中間ファイルは`.trident/build/game-module`へ分離されます。別のプロジェクトを開くと
EditorはそのプロジェクトのDLLへ切り替えるため、他ゲームのC++ Componentは混ざりません。
DLLがまだない新規プロジェクトも通常どおり開け、最初のビルドが完了すると自動で
Hot Reloadされます。プロジェクトDLLのDebug／Release構成は、起動中のEditorに自動追従します。

上級者向けに`TRIDENT_SCRIPT_WITH_SCHEMA`を使うと、公開値をInspectorへ型付きで表示できます。
現在は`bool`、`int`、`float`、`string`、`vec2`、`vec3`、`vec4`、`color3`、`color4`に対応し、
数値の`min`、`max`、`step`、表示名、ツールチップも指定できます。生のJSON編集欄は
「詳細設定 (Properties JSON)」として残るため、スキーマ外の値も編集できます。

```json
{
  "fields": [
    {
      "name": "speed",
      "displayName": "速度",
      "type": "float",
      "default": 5.0,
      "min": 0.0,
      "max": 20.0,
      "step": 0.1,
      "tooltip": "1秒あたりの移動量"
    }
  ]
}
```

サンプルの`samples/GameModule/SampleGameModule.cpp`には、従来どおり
`Sample.FloatingAccent`と`CoreRush.Director`も登録されています。

エンジンはDLLを`.trident-hot-reload`へシャドウコピーして読み込むため、
エディターを終了せずに`TridentGameModule`を再ビルドできます。更新は約0.5秒ごとに
検出され、実行中インスタンスをSerializeして破棄した後、新しいDLLで復元します。
Inspectorの「Moduleを再読み込み」から手動実行することもできます。
ゲームを書き出すと、現在のプロジェクトにあるGame Module DLLが配布フォルダーへ
`TridentGameModule.dll`としてコピーされます。プロジェクトDLLがない場合は、
C++ Componentなしのゲームとして書き出されます。

## コンポーネントの例

```cpp
auto& player = scene.CreateGameObject("Player");
player.GetTransform().position = { 0.0f, 0.0f, 0.0f };
player.AddComponent<Trident::MeshRendererComponent>(
    Trident::PrimitiveShape::Cube);
```

親子関係:

```cpp
auto& weapon = scene.CreateGameObject("Weapon");
weapon.SetParent(&player);
```

エンジン内蔵コンポーネントは`Trident::Component`を継承し、ゲーム固有コンポーネントは
上記のC++ Game Moduleへ登録します。

