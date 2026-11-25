# CLAUDE.md

このフォルダは、Mesh Avatar Studio で作ったアバターを使って「JAHIC ニュースチャンネル」向けの YouTube 解説動画（アバター1人のナレーション、または2人の掛け合い）を作るための作業場所です。git リポジトリは2つに分かれている:

- このフォルダ: `https://github.com/kounishi/avatar-gen-001`（**公開**、`main`）。`mesh-avatar-studio/`、音声・画像・動画・`.mavatar`、台本（`JAHIC_*.txt`）、`proj-*/` は `.gitignore` で除外。公開リポジトリなので、台本・セリフ・社外秘の情報はコミットしない。
- `mesh-avatar-studio/`: `https://github.com/kounishi/mesh-avatar-studio-lipsync`（**公開**、`main` ブランチ）。公開リポジトリなので、名前・コミット・コードに JAHIC などの案件名を入れない。本家は `upstream`（<https://github.com/shinshin86/mesh-avatar-studio>）。本家の更新は `git fetch upstream` → `git merge upstream/main` で取り込む。本家へは push しない。
- アバター（`mesh-avatar-studio/projects/`）はどちらにも入らない。メンバーへは `.mavatar` で渡し、元データは別途バックアップする。

ユーザーへの返答・ドキュメントは日本語で書く。

## 構成

| パス | 内容 |
|---|---|
| `mesh-avatar-studio/` | アバターエディタ本体（独立した git リポジトリ）。作業ルールは `mesh-avatar-studio/CLAUDE.md` → `AGENTS.md` に従う |
| `mesh-avatar-studio/projects/<アバター名>/` | アバター（`coresys-001`、`coresys-002`、`jahic-chan-001`、`etc-bouya-001`、`etc-girl`）。`coresys-002` は重ね絵 `hair_touch`（髪に手をやるしぐさ）付きで、書き出し時は `--idle-gain 0.5`（待機中の首・頭の揺れを半分）を付ける。重ね絵は `work/make-overlay.py` で `work/overlays/` に作り、`built/overlays/` にコピーする（build-layers の後に再コピー） |
| `mesh-avatar-studio/tools/` | `join-lines.py`（行音声の結合）、`render-lipsync.mjs`（口パク動画書き出し）など |
| `proj-NNN/` | 動画1本ぶんの素材。行ごとの音声 `番号_話者名_セリフ.wav`、結合結果 `joined.wav` と `joined.timeline.csv` |
| `JAHIC_*.txt` | 台本。`アバター名,セリフ` の CSV 形式（YMM 読み込み用） |
| `*.png` / `*.wav`（ルート直下） | アバターの元画像やサンプル音声 |
| `youtube動画作成手順.md` | 動画作成手順（OBS 録画、`render-lipsync`、2人掛け合いの流れ） |
| `導入手順.md` | 社内メンバーが自分の PC に環境を用意する手順（リポジトリ取得、`.mavatar` 取り込み） |
| `アプリ・Webページ組み込み手順.md` | `.mavatar` 書き出しと Web / React への組み込み手順 |

## 環境の作成（新しい PC に移すときも同じ）

このフォルダごとコピーしたあと、次を入れ直す。`projects/` は git 管理外なので、GitHub 経由ではなくフォルダごと移すこと。

| 必要なもの | バージョン | 用途・入れ方 |
|---|---|---|
| Node.js | 22.17 以上（現在 22.23） | エディタ・`render-lipsync`。<https://nodejs.org/> の LTS を入れる |
| uv | 現在 0.13 | Python スクリプト（`join-lines.py` など）の実行。`powershell -c "irm https://astral.sh/uv/install.ps1 \| iex"` |
| Python | 3.10 以上 | uv が必要に応じて自動で用意する（PC の Python 3.9 は使われない） |
| ffmpeg | 現在 8.1 | 動画の書き出し・音声の結合。`C:\ffmpeg\bin` に置き、PATH に追加する |
| Codex CLI | 任意 | アバターの目・口の差分画像を新しく描くときだけ必要 |

`mesh-avatar-studio` フォルダで初回のみ実行する:

```powershell
cd C:\Users\cores\source\repos\avatar-gen-001\mesh-avatar-studio
npm install
npx playwright install chromium   # render-lipsync・render-poses が使うブラウザ
```

- 社内ネットワーク（TLS の証明書エラー `invalid peer certificate: UnknownIssuer`）では、uv の実行前に `$env:UV_SYSTEM_CERTS = "1"` を設定する（恒久的にはユーザー環境変数に登録）。
- `node_modules` はコピーせず `npm install` で入れ直すほうが確実。
- 動作確認: `npm run dev` で <http://127.0.0.1:5173/> が開けること、`npm run render-lipsync -- projects/coresys-001 <音声> --duration 5` で動画ができること。
- `tools/speaker-timeline.py`（Whisper で音声1本から話者タイミングを作る試作）は初回に音声認識モデル（約 1.6 GB）をダウンロードする。GPU のある PC では `--device cuda` が使えるが未検証。行ごとの音声を使う通常の流れでは不要。

## アバター1人のナレーション動画の基本フロー（PowerShell、`mesh-avatar-studio` で実行）

話者が1人なのでタイミング表（`--timeline`）は不要。音声の声がすべてそのアバターの口パクになる。

```powershell
# A. ナレーション音声が1本のファイルの場合（出力: projects\<アバター>\video\<音声名>.mov）
npm run render-lipsync -- projects/coresys-001 ..\proj-NNN\narration.wav --no-half-eyes

# B. 行ごとの音声ファイルの場合: 先に結合してから書き出す（出力: projects\<アバター>\video\joined.mov）
uv run tools/join-lines.py ..\proj-NNN
npm run render-lipsync -- projects/coresys-001 ..\proj-NNN\joined.wav --no-half-eyes
```

- コマンド例には半目を出さない `--no-half-eyes` を付けている（半目も使う場合は外す）。しぐさ（重ね絵）を入れる場合は `--overlay hair_touch:12-15,40-43` のように秒数で指定する（`coresys-002` のみ。`youtube動画作成手順.md` セクション 7）。待機中の首・頭の揺れは `--idle-gain 0.5` のように小さくできる（`coresys-002` では付ける）。待機モーション（うとうと等）を止めたい場合は `--no-idle-motion` か `--calm` を付ける。
- 音声は**声だけ**のファイルを使う（BGM 入りだと声のない場面でも口が動く）。
- その後 DaVinci Resolve で背景動画（V1）・アバター（V2）・ナレーション音声（A1）を先頭を揃えて重ねる。アバター動画の音声はミュートするか、書き出し時に `--no-audio` を付ける。詳細は `youtube動画作成手順.md` のセクション 7 を参照。

## 2人掛け合い動画の基本フロー（PowerShell、`mesh-avatar-studio` で実行）

```powershell
# 1. 行ごとの音声を結合し、タイミング表を作る
uv run tools/join-lines.py ..\proj-NNN --map "コハク（ノーマル）=jahic-chan-001" --map "みちのくあいり（標準）=etc-bouya-001"

# 2. アバターごとに透過動画を書き出す（出力: projects\<アバター>\video\joined.<アバター>.mov）
npm run render-lipsync -- projects/jahic-chan-001 ..\proj-NNN\joined.wav --timeline ..\proj-NNN\joined.timeline.csv --no-half-eyes
npm run render-lipsync -- projects/etc-bouya-001  ..\proj-NNN\joined.wav --timeline ..\proj-NNN\joined.timeline.csv --no-half-eyes
```

- 掛け合いでは全員の書き出しに `--timeline` を必ず付ける（付け忘れると全員のセリフで口が動く。書き出し最初の `speaking ○% of the time` が 9 割近くなら付け忘れ）。`coresys-002` を使う場合は `--idle-gain 0.5` を足し、`--overlay` の秒数は coresys-002 が話している行の中から選ぶ。

その後 DaVinci Resolve で背景動画・2人のアバター・`joined.wav` を重ねて書き出す。詳細とオプションは `youtube動画作成手順.md` のセクション 7・8 を参照。

## 3人掛け合い動画の基本フロー（PowerShell、`mesh-avatar-studio` で実行）

2人のときと同じ仕組みで、`--map` と書き出しを3人分にする（4人以上も同様）。

```powershell
# 1. 行ごとの音声を結合し、タイミング表を作る（話者名=アバター名 を3つ指定）
uv run tools/join-lines.py ..\proj-NNN --map "話者1=アバター1" --map "話者2=アバター2" --map "話者3=アバター3"

# 2. アバターごとに透過動画を書き出す（3本。出力: projects\<アバター>\video\joined.<アバター>.mov）
npm run render-lipsync -- projects/アバター1 ..\proj-NNN\joined.wav --timeline ..\proj-NNN\joined.timeline.csv --no-half-eyes
npm run render-lipsync -- projects/アバター2 ..\proj-NNN\joined.wav --timeline ..\proj-NNN\joined.timeline.csv --no-half-eyes
npm run render-lipsync -- projects/アバター3 ..\proj-NNN\joined.wav --timeline ..\proj-NNN\joined.timeline.csv --no-half-eyes
```

- 同時に話す行は、ファイル名の話者名を `&` でつなぐ。3人同時は `話者1&話者2&話者3`、2人だけ同時なら `話者1&話者3` のように書ける（例: `031_話者1&話者2&話者3_こんにちは.wav`）。
- `join-lines.py` の最後に `Not mapped to a project:` が出たら、`--map` の話者名がファイル名と一致していない。
- DaVinci Resolve では背景（V1）、アバター3本（V2〜V4）、`joined.wav`（A1）を先頭を揃えて重ね、アバター動画の音声はミュートする。
- 書き出しは1人あたり約 20 分（3 分の音声）なので、3人で約 1 時間。まず `--duration 30` で3人分を試す。
- 3人同時の行は、テスト音声で結合とタイミング表の分け方まで確認済み。3人分の動画の本番書き出しは未検証。

## 注意点

- 音声ファイル名は `番号_話者名_セリフ.wav`。同時発話は話者名を `&` でつなぐ（3人以上も可）。
- `joined.timeline.csv` の話者名はアバターのプロジェクトフォルダ名と一致させる（違う場合は `--speaker`）。
- 書き出しは遅い（3 分の音声で 1 人あたり約 20 分）。まず `--duration 30` で試す。
- ffmpeg は `C:\ffmpeg\bin\ffmpeg.exe`。
- エディタは `npm run dev` → <http://127.0.0.1:5173/>。
- 公開用の `.mavatar` は元画像を含めない（`--no-source`）。
- ETC・JAHIC のロゴや文字を公開動画に出す場合は権利元への確認が必要。
