# リリース事故防止ルールブック

このファイルは、プレイリスト作成アプリを更新するときに同じ失敗を繰り返さないための確認ログです。

## 過去の失敗

### v1.1 / v1.1.1: macOSで「壊れている」扱いになった

原因:

- アプリ内部の実行ファイル名を日本語にしていた時期があり、配布時の署名・Gatekeeper判定が不安定になった。
- ad-hoc署名のみでApple Developer ID署名・公証はしていないため、初回起動時にGatekeeper警告が出る前提がある。

対応:

- `CFBundleExecutable` は必ず `PlaylistBuilder` にする。
- `Contents/MacOS` の中身は `PlaylistBuilder` 1つだけにする。
- 配布前に `codesign --verify --deep --strict --verbose=4` を必ず通す。

### v1.1.1: DMG内に古い実行ファイルが混入した

原因:

- DMG作成用フォルダを再利用し、古い `Contents/MacOS/プレイリスト作成アプリ` が残った。
- 署名後に余計なファイルが追加された形になり、`a sealed resource is missing or invalid` になった。
- この状態では「システム設定 > プライバシーとセキュリティ > このまま開く」が出ないことがある。

対応:

- リリースごとに `work/build-<version>` と `work/dmg-root-<version>` を新規作成する。
- 既存のDMG作成用フォルダを上書き再利用しない。
- DMG作成後、必ずマウントしてDMG内アプリを検証する。
- 既存アプリが古い更新URLをキャッシュする可能性があるため、必要に応じて直前バージョンのDMG/ZIPも正常ビルドへ差し替える。

## リリース前チェック

1. `Contents/MacOS` の中身を確認する。

```bash
ls -la <App>.app/Contents/MacOS
```

合格条件:

```text
PlaylistBuilder だけが存在する
```

2. Info.plistを確認する。

```bash
plutil -p <App>.app/Contents/Info.plist
```

合格条件:

```text
CFBundleExecutable = PlaylistBuilder
CFBundleShortVersionString = 配布するバージョン
CFBundleVersion = 前回より増えている
```

3. 署名検証を通す。

```bash
codesign --verify --deep --strict --verbose=4 <App>.app
codesign -dvvv <App>.app
```

合格条件:

```text
valid on disk
satisfies its Designated Requirement
Sealed Resources version=2
```

4. DMGを作成したら、必ずマウントして中身を再検証する。

```bash
hdiutil attach <DMG> -nobrowse -readonly
find /Volumes/<Volume>/<App>.app -maxdepth 3 -type f -print | sort
codesign --verify --deep --strict --verbose=4 /Volumes/<Volume>/<App>.app
hdiutil detach /Volumes/<Volume>
```

合格条件:

```text
古い実行ファイルが混入していない
DMG内の.appでもcodesign検証が通る
```

5. GitHubへpush後、実ダウンロードしてSHAを確認する。

```bash
curl --max-time 60 -L -o /tmp/playlist.dmg "<GitHub DMG URL>"
shasum -a 256 /tmp/playlist.dmg <local DMG>
```

合格条件:

```text
GitHubから落ちるDMGとローカル検証済みDMGのSHAが一致する
```

## 配布上の注意

- 現状はApple Developer ID署名・公証なしのad-hoc署名。
- そのため、初回起動時にmacOSのセキュリティ警告が出る可能性は残る。
- ただし、署名不整合や古いファイル混入による「壊れている」扱いは避ける。
- 外部や不特定多数へ配布する場合は、Apple Developer ID署名と公証を行う。
