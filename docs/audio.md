# オーディオ

効果音・BGM再生、3Dサウンド、ストリーミング、ミキサーバス。

[← ドキュメント一覧へ戻る](index.md)

## 1分でためす

1. 音を鳴らしたいGameObjectを選び、Inspectorの「コンポーネントを追加」から
   **オーディオソース**を追加します。
2. Asset BrowserのWAV／OGGファイルをInspectorへドラッグして割り当てます。
3. 「ゲーム開始時に自動再生」をONにしてPlayすると、音が鳴ります。

スクリプトから鳴らす場合（ジャンプ効果音の例）:

```cpp
#include "Trident/Trident.h"

class JumpSound final : public Trident::Script
{
public:
    void Start() override
    {
        // 同じGameObjectのオーディオソースを取得
        m_audio = GetComponent<Trident::AudioSourceComponent>();
    }

    void Update(float deltaTime) override
    {
        // Jump（既定はSpaceキー）を押した瞬間に1回だけ鳴らす
        if (m_audio != nullptr
            && Graphics().Input().WasPressed("Jump"))
        {
            m_audio->PlayOneShot();
        }
    }

private:
    Trident::AudioSourceComponent* m_audio{};
};

TRIDENT_SCRIPT(JumpSound);
```

BGMをコードから流す場合:

```cpp
auto& bgm = AddComponent<Trident::AudioSourceComponent>();
bgm.SetAudioPath("audio/bgm.ogg"); // assetsからの相対パス
bgm.SetLoop(true);
bgm.SetStreaming(true);  // 長い曲はメモリへ全展開せず再生
bgm.SetBus(Trident::AudioBus::Music);
bgm.SetVolume(0.6f);
bgm.Play();
```

`Play`（最初から再生）、`PlayOneShot`（重ねて1回）、`Pause`／`Resume`、`Stop`が
使えます。

## オーディオソースの設定

InspectorではAsset BrowserのWAV／OGGファイルをドラッグして音声を割り当て、
再生、One Shot、一時停止、再開、停止に加えて、音量、ピッチ、左右パン、ループ、
ゲーム開始時の自動再生を設定できます。
設定はシーンJSONへ保存され、GameObjectの複製にも引き継がれます。

長いBGMは「ストリーミング再生」をONにすると、圧縮データだけをメモリへ持ち、
再生しながらデコードします。ミキサーバス（Effects／Music）を選ぶと、
バスごとの音量をまとめて調整できます（効果音だけ小さくする等）。

## 3Dサウンド

3D音声を使う場合は、通常メインカメラへ「オーディオリスナー」を追加し、
AudioSourceの「3D空間オーディオ」を有効にします。音源とリスナーの位置・向きは
GameObjectのワールドTransformへ追従し、最小距離までは等音量、そこから線形に減衰して
最大距離で無音になります。3Dモードでは左右パンをTransformから自動計算します。

サンプルシーンはメインカメラにAudioListenerを持ち、右前方に置いた
`assets/audio/startup.wav`を3D音声として開始時に一度再生します。サンプル音源は
次のコマンドで再生成できます。

```powershell
powershell -ExecutionPolicy Bypass -File .\tools\GenerateSampleAudio.ps1
```

## よくあるつまずき

- **音が鳴らない** — 音声ファイルを割り当てたか、音量が0でないかを確認。
  自動再生がOFFの場合はスクリプトから`Play()`を呼ぶ必要があります。
- **3Dにしたら聞こえなくなった** — カメラ（または聞き手役のGameObject）に
  **オーディオリスナー**が必要です。最大距離が音源との距離より小さくても無音になります。
- **効果音が「ダダダ」と連打される** — `IsDown`（押している間ずっとtrue）ではなく
  `WasPressed`（押した瞬間の1フレームだけtrue）で鳴らします。
- **BGMでメモリを大量に使う** — 「ストリーミング再生」をONに。効果音のような
  短い音はOFFのまま（すぐ鳴らせる）が向いています。
- **音量調整のUIを作りたい** — 個別の音量ではなくバス音量
  （`Trident::AudioBus::Music`など）を操作すると、まとめて変えられます。
