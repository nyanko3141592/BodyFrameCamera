# PoseCam (WASM)

ブラウザのみでカメラ映像から単一人物の姿勢（33 ランドマーク）をリアルタイム推定し、`<canvas>` に骨格を描画する最小実装（MVP）。MediaPipe Tasks – Pose Landmarker (WASM, lite) を使用し、依存は CDN のみです。

## デモ / デプロイ（GitHub Pages）
- 公開例: `https://<your-username>.github.io/BodyFrameCamera/`
- 設定手順:
  1. ルート直下に `index.html` を配置して push
  2. GitHub → Repository → Settings → Pages
     - Build and deployment: Deploy from a branch
     - Branch: `main` / Folder: `/ (root)` を選択
  3. 数分で公開 URL が有効化（HTTPS 提供）

## 使い方
1. HTTPS 上でページを開く（GitHub Pages 推奨）
2. Start を押してカメラ権限を許可
3. Quality で 480p/720p、Flip で前面/背面を切替（対応端末）
4. 右上 HUD に FPS / Quality / 解像度 / 内部スケール / Facing / 推定状態 を表示

## 特徴（MVP）
- MediaPipe Pose Landmarker (WASM, lite) による単一人物推定
- `<canvas>` 2D 描画（DrawingUtils）
- Start/Stop、Quality(Low/Med)、Flip（前面/背面）
- 簡易アダプティブ制御（目標 ~24fps）で内部処理解像度を自動調整
- getUserMedia エラー種別に応じたユーザーメッセージ
- 完全クライアントサイド（映像/推定結果の外部送信なし）

## ローカル開発
`localhost` は安全なオリジンとして扱われるため HTTP でもカメラ権限が許可されます。

```bash
# 任意の簡易サーバでOK（例: Python）
cd /Users/takahashinaoki/Dev/Hobby/BodyFrameCamera
python3 -m http.server 8080
# → http://localhost:8080/
```

## ブラウザ対応（目安）
- 目標: Chrome / Edge 最新
- Safari / Firefox: WASM SIMD の可用性に依存（挙動確認を推奨）
- 非対応環境: 「HTTPS 必須」や権限拒否などのメッセージを表示

## ファイル構成
- `index.html`: 単一ファイルの実行本体（UI/推論/描画/HUD/エラー対応）
- `concept.md`: 仕様・設計ノート

## 依存
- CDN: `@mediapipe/tasks-vision@latest`（WASM）
- モデル: Google Hosted `pose_landmarker_lite.task`

## トラブルシュート
- 「HTTPS が必要」: GitHub Pages など HTTPS 上で開く（localhost は開発例外）
- 「カメラが見つからない」: Quality を変更、Flip で向きを切替
- 「使用中で読めない」: 他アプリがカメラを使用中の可能性
- 低 FPS: Quality を Low に、またはウィンドウを小さく

## プライバシー
完全クライアントサイド。映像や推定結果は外部送信しません（CDN から WASM/モデルを取得する通信のみ）。

## ライセンス
未指定。必要に応じて任意のライセンスを追加してください。
