# Mesh Avatar Studio のアバターをアプリ・Web ページに組み込む手順

対象アバター: `mesh-avatar-studio/projects/coresys-001`

流れは **アバターを 1 つのファイル（`.mavatar`）に書き出す → 配布されているライブラリで読み込む** の 2 段階。

---

## 1. アバターを `.mavatar` に書き出す

### エディタから

1. `mesh-avatar-studio` フォルダで `npm run dev` を実行し、<http://127.0.0.1:5173/> を開く。
2. **Open project** → `coresys-001` を開く。
3. 画面上部の **Export .mavatar** を押す。

### コマンドで

```powershell
cd C:\Users\cores\source\repos\avatar-gen-001\mesh-avatar-studio
npm run pack-avatar -- projects/coresys-001 --no-source -o coresys-001.mavatar
```

### 公開するときの注意

- 公開するサイトやアプリに載せる場合は、**元画像を含めない設定**にする。
  - エディタ: **Include source image** のチェックを外す
  - コマンド: `--no-source` を付ける
- 元画像がなくてもアバターは動く（ただし編集・作り直しはできなくなる）。

---

## 2. 組み込み方を選ぶ

| 方法 | 向いている場合 | ライブラリ |
|---|---|---|
| **A. HTML タグ** | 普通の Web ページ。いちばん簡単 | `mesh-avatar-element` |
| **B. React / Next.js** | React で作ったアプリ | `mesh-avatar-react` |
| **C. JavaScript で細かく操作** | 音声に合わせて口パク、表情の切り替えなど | `mesh-avatar` |

### A. HTML タグ（2 行で表示）

```html
<script type="module" src="https://cdn.jsdelivr.net/npm/mesh-avatar-element@0/dist/standalone.js"></script>
<mesh-avatar src="/avatars/coresys-001.mavatar" expression="smile"></mesh-avatar>
<style> mesh-avatar { width: min(100%, 320px); } </style>
```

- 背景は透明。大きさは CSS で幅を指定する（既定は縦横比 1:1）。
- 主な属性:

| 属性 | 内容 |
|---|---|
| `src` | `.mavatar` の URL |
| `expression` | 表情: `neutral`、`smile`、`shy`、`surprise`、`halfLidded`、`angry`、`sad`、`wink` |
| `emotion` | 感情タグ: `happy`、`sad`、`angry`、`surprised`、`relaxed`、`neutral` |
| `fit` | `contain`（全体を収める・既定）、`cover`（枠いっぱい） |
| `no-idle` | 付けると小さな待機モーション（うとうと・ため息など）を止める。まばたきと反応モーションは続く |
| `paused` | 付けるとアニメーションを止める |

npm で入れる場合は `npm install mesh-avatar-element` → JavaScript で `import 'mesh-avatar-element';`。

### B. React / Next.js

```sh
npm install mesh-avatar-react
```

```tsx
import { MeshAvatar } from 'mesh-avatar-react';

export function Character() {
  return <MeshAvatar src="/avatars/coresys-001.mavatar" expression="smile" style={{ width: 320 }} />;
}
```

- `onLoad={avatar => ...}` で読み込み後のアバターを受け取れる（C の操作がそのまま使える）。
- `onError` で読み込み失敗を受け取れる。

### C. JavaScript（音声に合わせて口パクする例）

```sh
npm install mesh-avatar
```

```html
<canvas id="avatar" width="720" height="720"></canvas>
<button id="play">再生</button>
```

```js
import { loadMeshAvatar } from 'mesh-avatar';

const avatar = await loadMeshAvatar(document.querySelector('#avatar'), '/avatars/coresys-001.mavatar');

// 音声を再生し、その大きさで口を動かす
const audio = new Audio('/voice/line1.mp3');
const ctx = new AudioContext();
const analyser = ctx.createAnalyser(); analyser.fftSize = 1024;
ctx.createMediaElementSource(audio).connect(analyser); analyser.connect(ctx.destination);
const buf = new Float32Array(1024);

audio.onplay = () => {
  avatar.setSpeaking(true);
  (function loop() {
    analyser.getFloatTimeDomainData(buf);
    const rms = Math.sqrt(buf.reduce((s, v) => s + v * v, 0) / buf.length);
    avatar.setVoiceLevel(Math.min(1, Math.max(0, (rms - 0.008) * 5)));
    if (!audio.paused) requestAnimationFrame(loop);
  })();
};
audio.onended = () => { avatar.setVoiceLevel(0); avatar.setSpeaking(false); };

// ブラウザの制限で、音声の再生はボタンを押したときなどユーザー操作の中で始める
document.querySelector('#play').onclick = async () => { await ctx.resume(); audio.play(); };
```

#### よく使う操作

| 操作 | 内容 |
|---|---|
| `avatar.setExpression('smile')` | 表情を切り替える（`'neutral'` で戻す） |
| `avatar.setEmotion('happy')` | 感情タグを付ける（`null` で戻す） |
| `avatar.setParameters({ angleX: 10, gazeX: 0.25 })` | 顔の向き・視線などを指定する（`{}` で解除） |
| `avatar.setAutoIdle(false)` | 小さな待機モーション（うとうと・ため息など、半目になるもの）を止める |
| `avatar.setAutoMotion(false)` | 反応モーション（うなずき・照れ・笑うなど）を止める |
| `avatar.setLipSync({ openness: 0.75 })` | 口の開き具合などを調整する（0〜1） |
| `avatar.destroy()` | アバターを片付ける（ページから外すとき） |

---

## 3. 用途別の補足

- **Web サイト**: `.mavatar` をサイトの公開フォルダ（例: `/avatars/`）に置く。別ドメインから読み込む場合は、サーバー側で CORS の許可設定が必要。
- **デスクトップアプリ（Electron、Tauri）**: 中身は Web ページなので、A〜C がそのまま使える。
- **スマホアプリ**: アプリ内の WebView で A〜C のページを表示する（WebGL が動く環境が必要）。
- **ネイティブ版**: Rust 製ランタイムもあるが実験的な扱い（`mesh-avatar-studio/native/README.md`）。今使うなら Web 版が無難。

## 4. 試作ページ（デモ）

`<mesh-avatar>` タグで組み込んだ試作ページを用意してある。

### 場所

`mesh-avatar-studio/projects/coresys-001/demo/`

| ファイル | 内容 |
|---|---|
| `index.html` | 試作ページ本体 |
| `coresys-001.mavatar` | アバター（元画像は含めていない） |
| `standalone.js` | アバター表示用ライブラリ（リポジトリからビルドしたもの） |

### 開き方

`index.html` をダブルクリックで直接開くとアバターを読み込めない。**必ずサーバー経由（http://〜）で開く。**

```powershell
cd C:\Users\cores\source\repos\avatar-gen-001\mesh-avatar-studio
npx vite projects/coresys-001/demo --port 5180
```

ブラウザで <http://127.0.0.1:5180/> を開く。止めるときはターミナルで `Ctrl + C`。

### できること

| 項目 | 内容 |
|---|---|
| 表情 | ふつう・笑顔・照れ・驚き・半目・怒り・悲しみ・ウインクを切り替える |
| 文字で口パク | 入力したひらがな・カタカナの母音に合わせて口が動く（音は出ない） |
| 音声ファイルで口パク | 選んだ音声を再生し、その声に合わせて口が動く（ファイルはブラウザの中だけで再生され、送信されない） |
| 表示 | 背景（透明・白・風景・緑）、大きさ、待機モーション・反応モーションのオン・オフ |

待機モーションと反応モーションの両方をオフにしても、顔・体のゆらぎ、呼吸、まばたきは続く（まばたきを止める設定はない）。

### 組み込みに必要な部分

`index.html` のうち、組み込みに必要なのは次の 2 行だけ（残りは試すためのボタン類）。

```html
<script type="module" src="./standalone.js"></script>
<mesh-avatar src="./coresys-001.mavatar"></mesh-avatar>
```

### 公開するとき

- `demo` フォルダごと Web サーバーに置けば、そのまま公開できる。
- 公開前に、アバターの絵とロゴの権利を確認する（「5. 権利について」）。

### アバターを更新したとき

エディタで調整・作り直しをしたら、`.mavatar` を書き出し直してデモフォルダに上書きする。

```powershell
cd C:\Users\cores\source\repos\avatar-gen-001\mesh-avatar-studio
Remove-Item projects/coresys-001/demo/coresys-001.mavatar
npm run pack-avatar -- projects/coresys-001 --no-source -o projects/coresys-001/demo/coresys-001.mavatar
```

（`pack-avatar` は既存のファイルを上書きしないため、先に古いファイルを削除する）

### ライブラリを作り直すとき

リポジトリを更新して `standalone.js` を新しくしたい場合:

```powershell
cd C:\Users\cores\source\repos\avatar-gen-001\mesh-avatar-studio
npm run build:runtime
npm run build:element
Copy-Item packages/element/dist/standalone.js projects/coresys-001/demo/ -Force
```

---

## 5. 権利について

- ライブラリ（`mesh-avatar` など）は MIT ライセンス。
- アバターの絵の権利は別。公開するサイトやアプリに使う場合は、**元画像の利用許可**と、「ETC」「JAHIC」の**ロゴの扱い**を確認しておく。

## 参考

- `mesh-avatar-studio/packages/runtime/README.md`（mesh-avatar）
- `mesh-avatar-studio/packages/element/README.md`（mesh-avatar-element）
- `mesh-avatar-studio/packages/react/README.md`（mesh-avatar-react）
- `mesh-avatar-studio/docs/format.md`（`.mavatar` の中身）
