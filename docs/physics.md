# 物理と衝突判定

Rigidbody、Collider、Raycast、CharacterController、物理マテリアル、CCD。

[← ドキュメント一覧へ戻る](index.md)

## 1分でためす（落ちる箱とジャンプ）

1. 床用GameObjectに**3Dボックスコライダー**を追加し、横に広げます
   （Rigidbodyは付けません＝動かない床になります）。
2. 箱用GameObjectにMeshRenderer（立方体）、**3Dボックスコライダー**、
   **リジッドボディ**を追加し、床の上へ置きます。
3. Playすると重力で落ちて床に載ります。ジャンプさせるには次のスクリプトを
   箱へ追加します。

```cpp
#include "Trident/Trident.h"

class JumpBox final : public Trident::Script
{
public:
    void Update(float deltaTime) override
    {
        // Space（既定のJump）で上向きに瞬間的な力を加えます
        if (Graphics().Input().WasPressed("Jump"))
        {
            auto* body =
                GetComponent<Trident::RigidbodyComponent>();
            if (body != nullptr)
            {
                body->AddForce(
                    { 0.0f, 6.0f, 0.0f },
                    Trident::ForceMode::Impulse);
            }
        }
    }

    void OnCollisionEnter(
        const Trident::CollisionEvent& event) override
    {
        // 何かに当たった瞬間に呼ばれます（着地判定などに）
    }
};

TRIDENT_SCRIPT(JumpBox);
```

人型キャラクターの移動は、自前でRigidbodyを操作するより
**Character Controller**（後述）を使うのが簡単です。WASDとSpaceが
最初から使えます。

## 物理と衝突判定

Inspectorの「コンポーネントを追加」から次を追加できます。

- 2Dボックスコライダー
- 3Dボックスコライダー
- 3Dカプセルコライダー
- 3Dスフィアコライダー
- 3D Convex Hullコライダー
- Mesh Collider（モデルの三角形メッシュ）
- リジッドボディ
- Character Controller

Rigidbodyは質量、速度、角速度、ローカル重心、移動／回転抵抗、重力、キネマティック、
回転軸ごとの固定、衝突検出モードを設定できます。力やトルクに加え、
`AddForceAtPosition`で重心から離れた位置へ力を加えられます。衝突解決も接触点の
速度とボックス形状から求めた慣性を使うため、崖際に偏って載った箱、斜めに接触した
OBB、摩擦で転がる物体が角運動を伴って反応します。

高速で移動する
弾丸などは`Continuous (CCD)`を選ぶと、移動距離とColliderの大きさに応じて1フレームを
最大32回へ自動分割し、薄いColliderのすり抜けを抑えます。通常の物体は負荷の小さい
`Discrete`を使用します。Colliderをトリガーにすると
衝突イベントのみ発生し、位置の押し戻しは行いません。
各Colliderの「摩擦係数」と「反発係数」はPhysics Materialとしてシーンへ保存されます。
摩擦は接触面に沿う速度を減速させ、反発は0で跳ね返らず、1で強く跳ね返ります。

```cpp
auto& body = object.AddComponent<Trident::RigidbodyComponent>();
body.SetMass(2.0f);
body.SetCenterOfMass({ 0.0f, -0.2f, 0.0f });
body.AddForceAtPosition(
    { 0.0f, 8.0f, 0.0f },
    worldHitPoint,
    Trident::ForceMode::Impulse);
```

`ForceMode`は継続的な`Force`／`Acceleration`と、瞬間的な`Impulse`／
`VelocityChange`に対応します。`CollisionEvent::point`からEnter／Stay時の
ワールド接触点も取得できます。

Box同士の面接触は、3Dでは最大4点、2Dでは接触辺の最大2点をContact Manifoldとして
生成します。Manifold内の法線Impulseを8回、同じBroad Phase候補に含まれる全接触ペアを
4回反復するため、上段の反力を下段の接触へ同じPhysics Substep内で伝え直せます。
十分低速で下から支持されたRigidbodyは自動的にSleepし、力、トルク、速度変更、
別の物体からの衝突でWakeします。C++の`Sleep()`／`WakeUp()`とInspectorの
「休止させる」／「起こす」から手動操作もできます。

物理演算は描画FPSと分離した固定60 Hzで実行されます。通常のアニメーションやUIは
`OnUpdate(deltaTime)`で経過時間を使い、Rigidbodyへ継続的な力を加える処理は
`OnFixedUpdate(fixedDeltaTime)`へ記述します。30／60／144 FPSのどの場合も1秒間に
60回の物理更新となるため、端末のフレームレートで速度や衝突結果が変化しにくい構成です。

Rigidbodyの「描画を補間」は既定で有効です。物理Transformは最新の確定状態を保持したまま、
描画時だけ直前と現在の位置を線形補間し、回転をQuaternion Slerpで補間します。
Mesh、Sprite、Light、Cameraと子GameObjectへ自動適用されるため、Compound Colliderの
子表示も親Rigidbodyへ滑らかに追従します。Teleportなど物理外でTransformを直接変更した
場合は履歴を同期し、古い位置から意図しない補間をしません。不要な物体はInspectorまたは
`RigidbodyComponent::SetInterpolate(false)`で無効化できます。
`Scene::PhysicsInterpolationAlpha()`と`GameObject::InterpolatedWorldMatrix()`から
描画補間率／補間済み行列を独自Rendererでも利用できます。

```cpp
void OnFixedUpdate(float fixedDeltaTime) override
{
    auto* body = Owner().GetComponent<Trident::RigidbodyComponent>();
    body->AddForce(
        { 0.0f, 0.0f, 8.0f },
        Trident::ForceMode::Acceleration);
}
```

3Dボックスは回転を含むOBBとして15軸SATで判定します。Capsule Colliderと
Sphere ColliderはBox／Capsule／Sphere間のすべての組み合わせに対応し、
キャラクター、縦長の物体、ボール状の物体へ軽い当たり判定を設定できます。
Scene Viewのデバッグ表示も実際の回転ボックス、カプセル、スフィア形状に追従します。

Convex Hull ColliderはInspectorで頂点を直接編集する任意点群コライダーで、
GJK（交差判定）とEPA（めり込み量・法線算出）による汎用アルゴリズムで
Box／Capsule／Sphere／Hullのすべての組み合わせに対応します。クレーンの
ブレードや傾いた岩など、既存の基本形状では表現しづらい凸多面体形状に使えます。

Unityと同様に、Rigidbodyを親GameObjectへ置き、Box／Capsule／Sphere Colliderを
子GameObjectへ配置するとCompound Colliderとして動作します。子の位置・回転・大きさで
ひとつの剛体形状を組み立てられ、同じ親Rigidbodyに属するCollider同士は自己衝突しません。
衝突による移動・回転・Sleepは親Rigidbodyへ適用されます。

Character ControllerはTransformの位置を足元として扱い、入力移動、重力、接地判定、
ジャンプ、壁沿い移動、段差越えに対応します。既定では`MoveHorizontal`、
`MoveVertical`、`Jump` Actionを使用します。スクリプトからは`Move()`と`Jump()`でも
操作でき、Inspectorには接地状態と垂直速度がリアルタイム表示されます。

SceneにはColliderを調べるPhysics Queryがあります。

```cpp
Trident::PhysicsHit hit;
if (scene.Raycast(ray, 100.0f, hit))
{
    auto* object = hit.gameObject;
}

const auto nearby = scene.OverlapSphere(center, 2.0f);
```

`Raycast`、距離順の`RaycastAll`、`SphereCast`、`OverlapBox`、`OverlapSphere`を利用でき、
Layer Mask、Triggerを含めるか、無視するGameObjectをクエリごとに指定できます。

独自コンポーネントでは次のイベントをオーバーライドできます。

```cpp
void OnCollisionEnter(const Trident::CollisionEvent& event) override;
void OnCollisionStay(const Trident::CollisionEvent& event) override;
void OnCollisionExit(const Trident::CollisionEvent& event) override;
```

2Dは軽量なワールドAABB、3DのNarrow PhaseはOBB／Capsule／Sphere判定です。
Physics QueryはBox、Capsule、Sphereを対象にし、現在は高速な外接Bounds判定を使用します。

`Joint`コンポーネントでは接続先GameObjectと双方のローカルアンカーを指定できます。

- `Fixed`: アンカー位置、速度、角速度を固定
- `Hinge`: アンカー位置と軸外の回転を固定し、指定軸だけ回転
- `Spring`: 自然長、強さ、減衰を使って2物体を接続
- `接続した物体同士を衝突`: 無効時は接続ペアのCollider判定を省略

```cpp
auto& body = scene.CreateGameObject("Door");
body.AddComponent<Trident::RigidbodyComponent>();
auto& hinge = body.AddComponent<Trident::JointComponent>(
    Trident::JointType::Hinge,
    frame.Id(),
    DirectX::XMFLOAT3{ -0.5f, 0.0f, 0.0f },
    DirectX::XMFLOAT3{ 0.5f, 0.0f, 0.0f });
hinge.SetUseLimits(true);
hinge.SetLimits({ -75.0f, 75.0f });
hinge.SetUseMotor(true);
hinge.SetMotor({
    45.0f,  // 目標角速度（度/秒）
    20.0f   // 最大トルク
});
```

Fixed Jointは位置アンカーと角速度を固定し、Hingeは位置アンカーを拘束しながら
初期姿勢を0度として最小／最大角度の範囲で回転します。モーターは目標角速度まで
最大トルクの範囲で加速し、角度制限の外側へは押し続けません。Springは質量差を
反映して伸縮します。Joint Constraintは各Physics Substepで4回反復されるため、
複数のJointを連結したときもアンカー誤差が伝播しにくくなっています。

Broad Phaseでは2D・3D Colliderを均一グリッドへ登録し、同じセルを共有する
ColliderだけをNarrow Phaseの候補にします。複数セルにまたがるColliderのペアは
一度だけ処理され、広大な床など4096セルを超えるColliderは取りこぼしを防ぐため
安全な全候補比較へ自動的に切り替わります。

Inspectorの「シーン環境 → 物理 Broad Phase」ではセルサイズを0.25～100の範囲で
調整できます。再生中はCollider数、候補ペア数、実際のNarrow Phase回数、
使用セル数、接触数をリアルタイム表示します。セルサイズはシーンJSONの
`physics.broadPhaseCellSize`へ保存されます。

## よくあるつまずき

- **床をすり抜けて落ちる** — 床か物体のどちらかに**Colliderが無い**のが
  ほとんどの原因です。Scene ViewでColliderは緑の枠で表示されるので目で確認できます。
  高速で動く物体のすり抜けは、Rigidbodyの衝突検出を`Continuous (CCD)`へ。
- **押しても動かない** — 「キネマティック」がONだと物理の影響を受けません。
  また静止し続けた物体はSleepします（力を加えれば自動で起きます）。
- **Rigidbody付きなのにTransformを直接動かしたら挙動が変** — Rigidbodyを
  持つ物体は`AddForce`／`SetVelocity`で動かします。ワープさせたい場合だけ
  Transformを直接書き換えます（補間履歴は自動同期されます）。
- **OnCollisionEnterが呼ばれない** — ColliderがTriggerになっていると
  `OnTriggerEnter`側へ届きます。両方すり抜けたい場合はTrigger、
  ぶつけたい場合は通常Colliderです。
- **ジャンプや加速をUpdateに書いたらフレームレートで変わる** — 継続的に力を
  加える処理は固定60Hzの`FixedUpdate`へ。押した瞬間の`Impulse`は`Update`でも
  問題ありません。

