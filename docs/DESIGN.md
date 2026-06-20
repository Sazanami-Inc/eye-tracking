# 設計指針

## 1. プロジェクト概要

### 目的
小学生が iPad 上で数学問題を解く際の視線を全記録し、以下を明らかにする。

- **どこに集中しているか** — 問題のどの要素（数字・記号・図）を注視しているか
- **どこで詰まるか** — 視線の迷い・往復が認知的負荷のシグナルになる
- **正誤との相関** — 視線パターンと正解/不正解の関係

最終目標: 収集データで fine-tuning した **数学教育特化型モデル** の構築。

### 参照プロジェクト
- [PINTO0309/screen-eye-tracking](https://github.com/PINTO0309/screen-eye-tracking) — ONNX/OpenCV による軽量視線推定（Python デスクトップ版）

---

## 2. フェーズ設計

```
Phase 1 (PoC)     Phase 2 (データ収集)     Phase 3 (モデル構築)
──────────────    ─────────────────────    ────────────────────
TypeScript         TypeScript + Python       Python
iPad Safari        実際の小学生環境          HuggingFace / PyTorch
WebGazer.js        精度改善・運用化          特化型モデル評価
```

---

## 3. PoC アーキテクチャ（TypeScript）

### 技術選定

| 層 | 技術 | 選定理由 |
|---|---|---|
| 視線取得 | [WebGazer.js](https://webgazer.cs.brown.edu/) | iPadブラウザで動作、カメラ1本で視線推定 |
| フレームワーク | Next.js (App Router) | ルーティング・API Routes が一体 |
| 型安全API | tRPC | フロント↔バック間の型共有 |
| DB | SQLite (Drizzle ORM) | PoC段階のローカル保存 |
| 問題レンダリング | KaTeX | 数式のブラウザ表示 |

### データフロー

```
iPad カメラ
    │
    ▼
WebGazer.js  ──── 推定視線座標 (x, y, timestamp) ────▶ GazeCollector
                                                           │
問題表示コンポーネント ──── 要素ごとの Bounding Box ────▶ GazeMapper
                                                           │
                                                           ▼
                                                    GazeEvent (構造化)
                                                           │
                                                           ▼
                                                    API Route → SQLite
```

### WebGazer 精度の現実

WebGazer はキャリブレーション後で誤差 **±100〜150px** 程度（画面サイズ依存）。  
iPad 12.9inch（2732×2048px）なら「どの要素を見ているか」の象限レベルは取れる。  
小学生の細かい視線を拾う本番では Tobii や Apple Vision SDK の検討が必要。

---

## 4. データ設計

### 収集するイベント

```typescript
// packages/schema/src/types.ts

type GazePoint = {
  x: number;           // 画面左端からのpx
  y: number;           // 画面上端からのpx
  timestamp: number;   // Unix ms
  confidence: number;  // WebGazer信頼度 0.0–1.0
};

type ElementHit = {
  elementId: string;   // "problem-title" | "number-3" | "operator-plus" など
  boundingBox: { top: number; left: number; width: number; height: number };
  dwellMs: number;     // この要素を連続して注視した時間
};

type Session = {
  sessionId: string;
  studentId: string;   // 匿名化済みID
  problemId: string;
  startedAt: number;
  endedAt: number | null;
  answer: string | null;
  isCorrect: boolean | null;
  gazePoints: GazePoint[];
  elementHits: ElementHit[];
};
```

### 派生指標（前処理で計算）

| 指標 | 定義 | 意味 |
|---|---|---|
| `totalDwellMs` | 各要素への注視時間合計 | どこに時間をかけたか |
| `saccadeCount` | 素早い視線移動の回数 | 探索行動の量 |
| `regressionCount` | 一度見た要素への視線戻り回数 | 理解の詰まり |
| `firstFixation` | 最初に注視した要素ID | 問題を読む入口 |
| `scanPathLength` | 視線軌跡の総距離 | 認知的コスト |

---

## 5. LLM 学習パイプライン（Python、Phase 3）

### アーキテクチャ

```
収集データ (SQLite / JSON)
    │
    ▼ 前処理
Python (pandas, numpy)
  - ノイズ除去（confidence < 0.3 を除外）
  - 正規化（問題サイズ差を吸収）
  - セッション単位でのラベル付け
    │
    ▼ 特徴量エンジニアリング
視線シーケンス → トークン化
  例: ["look:number-3(450ms)", "saccade:operator-plus", "look:number-5(200ms)", "answer:8(correct)"]
    │
    ▼ fine-tuning
HuggingFace Transformers
  ベースモデル: Qwen2.5-Math-1.5B など数学特化の小型モデル
  学習目的: 視線シーケンスから「次にどこを見るべきか」「どこで詰まるか」の予測
    │
    ▼ 評価
  - 予測精度（視線パターン予測）
  - 教育的有用性（詰まりポイント検出率）
```

### ベースモデル候補

| モデル | パラメータ | 特徴 |
|---|---|---|
| Qwen2.5-Math-1.5B | 1.5B | 数学特化、軽量 |
| DeepSeek-Math-7B | 7B | 数学推論が強い |
| GPT-4o (APIのみ) | — | ベースライン比較用 |

---

## 6. 実装ロードマップ

### Phase 1: PoC（〜4週間）
- [ ] Next.js プロジェクトセットアップ
- [ ] WebGazer.js 統合・キャリブレーション画面
- [ ] 数学問題表示コンポーネント（KaTeX）
- [ ] 視線座標 → 要素マッピング
- [ ] セッションデータ保存（SQLite）
- [ ] 簡易ダッシュボード（ヒートマップ表示）

### Phase 2: データ収集（〜3ヶ月）
- [ ] 倫理審査・保護者同意フロー
- [ ] 匿名化パイプライン
- [ ] 問題バンク設計（学年・単元別）
- [ ] 精度改善（キャリブレーション最適化）
- [ ] データ品質チェック自動化

### Phase 3: モデル構築（データ収集後）
- [ ] 前処理パイプライン（Python）
- [ ] 視線シーケンスのトークナイザ設計
- [ ] ベースモデル選定・fine-tuning
- [ ] 評価フレームワーク設計
- [ ] 推論API構築

---

## 7. 技術的リスクと対策

| リスク | 対策 |
|---|---|
| WebGazer の精度不足 | Phase 2でTobii or iPadのARKit視線API移行を検討 |
| 小学生のキャリブレーション困難 | ゲーム形式のキャリブレーションUI |
| データ量不足 | Transfer Learning + Data Augmentation |
| 個人情報（顔画像） | オンデバイス処理、座標のみ送信、顔画像は保存しない |
| 倫理的課題 | 保護者同意、データ最小化、匿名化を設計に組み込む |

---

## 8. 参考文献・先行研究

- EyeLayer (2026) — 開発者視線データでLLMのコード要約改善
- HumanLLM pipeline (2026) — 視線データをCodeT5 fine-tuningに注入
- Frontiers in Education (2024) — 数学教育への視線追跡適用64本の論文調査
- PINTO0309/screen-eye-tracking — ONNXベース軽量視線推定の参照実装
