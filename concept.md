# PoseCam (WASM) — シングル人物・姿勢検出 実装方針（軽量・迅速）

## 0. ゴール / 非ゴール

**ゴール**

- ブラウザのみで、カメラ映像から**単一人物**の姿勢（33 点前後のランドマーク）を**リアルタイム**に推定し、キャンバス上に骨格を可視化する。
- **軽量・依存最小**・**実装迅速**。GPU なしでも実用 FPS（目安: 15–30fps）を狙う。

**非ゴール**

- 複数人検出、3D SMPL 推定、高度なトラッキング/ID 永続化、動作認識。

---

## 1. 技術選定（最短ルート）

- **モデル/ランタイム**: Google **MediaPipe Tasks – Pose Landmarker (Web)**

  - 実行: **WASM (SIMD)** を既定。対応環境では自動で最適版を使用。
  - モデル: `pose_landmarker_lite.task`（軽量）を採用。

- **映像入出力**: `getUserMedia()` + `<video>`
- **可視化**: `<canvas>` 2D コンテキスト（`DrawingUtils` 併用）
- **並列化**: v1 はメインスレッドで実装。必要に応じて **Web Worker** へ切り出し。
- **解像度/スケーリング**: 入力を **低解像度（例: 480p）** に固定し、負荷に応じて動的ダウンサンプル。

> 代替（任意）: ONNX Runtime Web + WebGPU は将来の高速化候補。v1 では採用しない。

---

## 2. 最小プロダクト要件（MVP）

- カメラ許可ダイアログ → 映像取得。
- Pose 推定（単体） → キーポイント・コネクタ描画。
- 画面サイズに合わせてキャンバスをリサイズ。デバイス回転に追従。
- **FPS/負荷に応じた解像度制御**（簡易アダプティブ）

  - 目標 FPS 未満が連続したら、内部処理用のスケール係数を低下。

- **エラーハンドリング**（権限拒否、カメラ無、HTTPS でない等）。

---

## 3. アーキテクチャ

```
┌────────────┐      ┌──────────────────┐
│ Camera (getUserMedia) ├─────▶│ Pose Landmarker (WASM) │
└────────────┘      └──────────────────┘
        │ frame (HTMLVideoElement)              │ landmarks
        ▼                                       ▼
   <canvas> draw ──────────────────────────▶  DrawingUtils / custom renderer
```

- **Main Loop**: `requestAnimationFrame` ベース。毎フレーム: 1) video→pose 推定, 2) 背景描画, 3) 骨格描画。
- **Timing**: `detectForVideo(video, timestampMs)` を使用。
- **State**: 直近ランドマーク、fps 移動平均、内部スケール係数。

---

## 4. UI/UX 仕様（v1）

- 画面: **全画面プレビュー**（ミラー表示）。右上にステータス HUD（FPS / 解像度 / 推定有無）。
- ボタン:

  - **Start/Stop**（カメラ取得/停止）
  - **Quality**: Low/Med（既定: Low=480p）

- アクセシビリティ: 権限拒否時の明確なメッセージと再試行導線。

---

## 5. パフォーマンス方針

- **入力縮小**: `drawImage(video, 0,0,w*h)` 前に内部オフスクリーンで縮小。
- **アダプティブ制御**: `targetFps=24` 付近。3 秒間の平均 FPS が下回れば `scale *= 0.85`、上回ればゆっくり戻す。
- **描画の最小化**: 背景は `drawImage` のみ、骨格はパスをまとめて描く。
- **メモリ**: バッファ再利用、不要なオブジェクト割当を避ける。

---

## 6. セキュリティ/プライバシー

- **完全クライアントサイド**。映像/推定結果を外部送信しない。
- TLS(HTTPS) 必須。`localhost` は開発例外。
- 許可取り消しとデバイス列挙の UI を用意（将来）。

---

## 7. ブラウザ対応

- 目標: **Chrome/Edge 最新**。Safari/Firefox は挙動確認のみ（WASM SIMD の可用性に依存）。
- 非対応環境: 明示メッセージ（"Your browser does not support required features"）。

---

## 8. モジュール構成（想定）

- `app.ts`：エントリ。UI 初期化、ループ管理。
- `camera.ts`：カメラ取得/リサイズ、デバイス切替（将来）。
- `pose.ts`：FilesetResolver、PoseLandmarker ラッパ、`init() / detect(video, ts)`。
- `renderer.ts`：背景・骨格描画、HUD。
- `perf.ts`：FPS/EMA、アダプティブ制御。
- `types.ts`：型定義。

---

## 9. 主要フロー（疑似コード）

```ts
// app.ts
await ui.init();
const cam = await camera.start({ width: 640, height: 480, facingMode: "user" });
const pose = await poseMod.init({ model: "lite" });

let scale = 0.75; // 内部処理解像度スケール
let emaFps = 0;

function loop(t) {
  perf.begin();
  const landmarks = pose.detect(cam.video, t);
  renderer.draw(cam.video, landmarks, { scale, fps: perf.fps() });
  emaFps = perf.ema(emaFps);
  scale = perf.autoScale(emaFps, scale); // 目標 FPS に合わせて微調整
  requestAnimationFrame(loop);
}
requestAnimationFrame(loop);
```

---

## 10. エラーハンドリング

- **権限拒否/デバイス無し**: エラーを UI に表示、設定リンク提示。
- **HTTPS でない**: 警告 + 早期 return。
- **モデル/wasm 読込失敗**: リトライ、CDN ミラーを用意（任意）。

---

## 11. テレメトリ（任意）

- 送信しない前提。ただし開発ビルドでは `console.table` で FPS/スケール推移を確認。

---

## 12. 将来拡張

- **Web Worker** で推論を分離（`OffscreenCanvas` 併用）。
- **WebGPU** 実行（ONNX Runtime Web）で高 FPS 化。
- **角度/姿勢分類**（例: 深いスクワット検出）。
- **複数人**（MoveNet MultiPose 等）
- **録画/リプレイ**（WebCodecs/MediaRecorder）

---

## 13. 参考実装スニペット（MVP）

```html
<video id="cam" autoplay playsinline></video>
<canvas id="viz"></canvas>
<script type="module">
  import {
    FilesetResolver,
    PoseLandmarker,
    DrawingUtils,
  } from "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest";

  const video = document.getElementById("cam");
  const canvas = document.getElementById("viz");
  const ctx = canvas.getContext("2d");

  // カメラ
  const stream = await navigator.mediaDevices.getUserMedia({
    video: { width: 640, height: 480, facingMode: "user" },
  });
  video.srcObject = stream;
  await video.play();
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;

  // Pose (WASM)
  const vision = await FilesetResolver.forVisionTasks(
    "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm"
  );
  const landmarker = await PoseLandmarker.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath:
        "https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_lite/float16/1/pose_landmarker_lite.task",
    },
    runningMode: "VIDEO",
    numPoses: 1,
  });

  const draw = new DrawingUtils(ctx);
  let scale = 0.75,
    ema = 0;
  const target = 24;

  async function loop() {
    const t = performance.now();
    const res = await landmarker.detectForVideo(video, t);
    // 背景
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    // 骨格
    if (res.landmarks?.length) {
      const lm = res.landmarks[0];
      draw.drawLandmarks(lm);
      draw.drawConnectors(lm, PoseLandmarker.POSE_CONNECTIONS);
    }
    // 簡易 HUD（FPSは setInterval 側で更新でも可）
    requestAnimationFrame(loop);
  }
  loop();
</script>
```

---

## 14. AI コード生成用プロンプト（貼り付け用）

> 目的: ブラウザのみで単一人物の姿勢をリアルタイム推定し、キャンバスに骨格可視化する MVP を作る。MediaPipe Pose Landmarker (WASM, lite) を使い、入力は 480p 程度に抑える。描画は `<canvas>`。アダプティブに解像度を下げて FPS を維持する。依存は CDN のみ、ビルド不要の 1 ファイル構成で開始。UI は Start/Stop と Quality(Low/Med)。エラーハンドリングと権限拒否の文言も付ける。

---

## 15. 受け入れ基準（Definition of Done）

- HTTPS 環境でページを開くと、カメラ許可後に**骨格の点/線**が表示される。
- 通常ノート PC で **15fps 以上**を維持（Low 品質時）。
- 権限拒否/デバイス無し/読み込み失敗時に**ユーザ向けメッセージ**が表示される。
- 外部送信なし（ネットワークはモデル/wasm の取得のみ）。
