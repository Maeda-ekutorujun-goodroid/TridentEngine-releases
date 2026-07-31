# パッケージ置き場

TridentEditorの「ウィンドウ」→「パッケージ」タブが参照する、公式パッケージの置き場です。

- `index.json` — パッケージ一覧。エディターはここを読み込みます
- `<name>-<version>.zip` — パッケージ本体（C++スクリプト・Prefab・アセット一式）

## index.jsonの形式

```json
{
  "format": "TridentPackageIndex",
  "version": 1,
  "packages": [
    {
      "name": "camera-follow",
      "displayName": "カメラ追従（3人称／2D）",
      "description": "ターゲットを滑らかに追うカメラ。",
      "author": "公式",
      "version": "1.0",
      "minimumEngineVersion": "2026.7.31.2",
      "downloadUrl": "https://raw.githubusercontent.com/Timiratz/TridentEngine-releases/main/packages/camera-follow-1.0.zip",
      "sizeBytes": 2048
    }
  ]
}
```

- `name` は英小文字・数字・`-`・`_`のみ（インストール先フォルダー名になります）
- `downloadUrl` はこのリポジトリ配下のURLのみエディターが許可します
- Zipの中に `package.json`（name/version等）を含めると、それがインストール情報として使われます（無ければ一覧から自動生成）
