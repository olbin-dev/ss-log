# マクロ → 産業翻訳エンジン (Macro Engine) MVP & ロードマップ

本ドキュメントは、マクロ経済指標の変動を分析し、各産業セクターへの影響度を決定論的にスコアリング・ランク付けする**「マクロ → 産業翻訳エンジン (Macro Engine)」**の開発ドキュメントおよび将来のロードマップをまとめたものです。

---

## 1. プロジェクトの概要とディレクトリ構成

このツールは、FRED（セントルイス連邦準備銀行）などの外部APIからリアルタイムにマクロ経済データを取得し、あらかじめ定義された影響度マトリックス（ルール）に基づいて、17の主要産業セクターのスコアを算出します。

ソースコードは `/Users/p/FIA/translation_engine/` にて管理されています。

```
/Users/p/FIA/translation_engine/
├── README.md                 # プロジェクトの概要とセットアップガイド
├── requirements.txt          # 依存パッケージ (pandas, numpy, pytest, streamlit, plotly)
├── main.py                   # 統合CLIデモ実行スクリプト
├── app.py                    # StreamlitベースのインタラクティブWebダッシュボード
└── src/
    ├── __init__.py
    ├── schema.py             # 17セクター、サブセクターマッピング、影響度マトリックスの定義
    ├── detector.py           # 金利、CPI、原油、円相場、GDPなどの変動・重篤度検知 (EventDetector)
    ├── engine.py             # 加重合計スコアリングおよびセクター順位付け (RuleEngine)
    └── ingestion.py          # FRED APIデータ取得 (オフラインCSVフォールバック付き)
```

---

## 2. コア実装ロジック

### A. 静的スキーマ定義 (`src/schema.py`)
- **17の主要セクター**: `bank` (銀行), `insurance` (保険), `real_estate` (不動産), `construction` (建設), `automobile` (自動車), `machinery` (機械), `electronics` (電機), `semiconductor` (半導体), `trading_company` (商社), `retail` (小売), `shipping` (海運), `energy` (エネルギー), `materials` (素材), `pharma` (医薬品), `telecom` (通信), `defense` (防衛), `railway` (鉄道)。
- **サブセクターマッピング (`SUBSECTOR_TO_SECTOR_MAP`)**: 細かなインジケーターや影響先（例: `housing`, `gpu_related`, `retail_import`）を、上記の17主要セクターへ適切に紐づけます。
- **影響度マトリックス (`IMPACT_MATRIX`)**: 各マクロイベントが特定のセクターに与えるベース影響度（例: `+50`, `-30`）を定義しています。

### B. イベント検知と重篤度マッピング (`src/detector.py`)
主要な指標の期間変化率（金利はパーセントポイント、原油や為替はパーセント変化）を元に、自動でイベントと重篤度（Severity: 1〜3）を検知します：
- **10年債利回り変化 (30日間)**: $+0.25\%$, $+0.5\%$, $+1.0\%$ の上昇でそれぞれ重篤度 `1`, `2`, `3` の `long_rate_spike` をトリガー。下落時は `long_rate_drop` をトリガー。
- **CPIの前年比 (YoY)**: $2\%$, $3\%$, $5\%$ を超える水準でそれぞれ重篤度 `1`, `2`, `3` の `inflation_high` をトリガー。低インフレ時は `inflation_low`。
- **WTI原油価格 (30日間)**: $\pm 10\%$, $\pm 20\%$, $\pm 40\%$ 以上の変動で `oil_price_spike` / `oil_price_crash` をトリガー。
- **ドル円為替レート (30日間)**: $\pm 2\%$, $\pm 5\%$, $\pm 10\%$ 以上の変動で `yen_weakening` (円安) / `yen_strengthening` (円高) をトリガー。

### C. ルールスコアリング数式 (`src/engine.py`)
検知されたイベントごとに、静的マトリックスからベース影響度を取得し、以下の式でセクターごとの累積スコアを算出します：

$$\text{Industry Score} = \sum (\text{Base Impact Weight} \times \text{Severity})$$

その後、全体の絶対値最大スコアを基準に正規化（Normalized Score）を行います：

$$\text{Normalized Score} = \frac{\text{Score}}{\max_{s \in \text{Sectors}}(|\text{Score}_s|)}$$

正規化されたスコア（-1.0 〜 +1.0）により、マクロ環境における各産業の相対的な有利・不利が可視化されます。

---

## 3. テストと動作結果

### A. 自動テスト (`pytest`)
APIのモック経路、データフレーム処理、オフセット計算などを検証するテスト群が作成されており、すべてパスしています。

```bash
platform darwin -- Python 3.14.5, pytest-9.0.3
collected 9 items

tests/test_ingestion.py ....                                             [ 44%]
tests/test_translation.py .....                                          [100%]

============================== 9 passed in 0.27s ===============================
```

### B. CLIデモ実行ログ (`python main.py`)
FRED APIキーが設定されていない場合でも、公開CSVファイルから最新の金利・CPI・為替・原油・GDPデータを自動でダウンロードし、スコアを計算します。以下は実行結果のサンプルです：

```
--- TRIGGERED MACRO EVENTS ---
🟢 [MODERATE] inflation_high            | High inflation detected. YoY CPI is 3.8%
🟢 [MODERATE] gdp_growth_accelerating   | GDP growth accelerating. YoY: 5.9%
🟢 [MINOR] ai_investment_boom        | Manual event trigger: ai_investment_boom (severity: 1)
🟢 [MINOR] semiconductor_capex_cycle | Manual event trigger: semiconductor_capex_cycle (severity: 1)
------------------------------

===============================================================================================
RANK | CORE SECTOR        | SCORE    | NORMALIZED | CONTRIBUTING FACTORS & IMPACTS
===============================================================================================
   1 | semiconductor      | +200.0 |    +1.0000 | ai_investment_boom (+50 * 1 = +50), semiconductor_capex_cycle (+50 * 1 = +50)...
-----------------------------------------------------------------------------------------------
   2 | energy             | +140.0 |    +0.7000 | inflation_high (+40 * 2 = +80), inflation_high (+30 * 2 = +60)
-----------------------------------------------------------------------------------------------
   ...
  17 | retail             | -100.0 |    -0.5000 | inflation_high (-30 * 2 = -60)...
```

---

## 4. Webダッシュボードの機能 (`app.py`)

StreamlitとPlotlyを使用したWebインタラクション環境が構築されています：
- **主要KPI表示**: 最新の10年債金利、CPI前年比、WTI原油価格、ドル円レート、GDP成長率および30日変動デルタを表示。
- **動的オーバーライド (サイドバー)**: 「AI投資ブーム」「半導体設備投資サイクル」「防衛費拡大」「中国経済減速」などの定性的・主観的マクロイベントのスライダーとチェックボックスを装備。
- **産業別スコアチャート**: Plotlyにより極性（好影響は緑、悪影響は赤、中立はグレー）を動的にカラーリングした横棒グラフ。
- **計算監査グリッド**: どのマクロ要因がどのウェイトで計算されたかを透明に追跡可能。

---

## 5. 将来の開発ロードマップ (Phases 1-5)

StreamlitによるWebモックアップ（フェーズ1）の完了を受け、本ツールは本格的なデスクトップアプリケーション（PySide6）へ向けて以下の段階で進化させます。

### フェーズ 2: 個別株評価とスクリーニング (マクロ + ミクロ結合)
マクロセクターのスコアを個別の銘柄評価に接続します：
- yfinanceやJ-Quantsから割安度（PER, PBR, 配当利回り）および収益性（ROE, 自己資本比率）を取得。
- 複合評価モデルの導入：
  $$\text{Ticker Score} = \text{Normalized Sector Score} \times w_{\text{macro}} + \text{Valuation Score} \times w_{\text{value}}$$
- **BUY / WATCH / AVOID** のスクリーナー判定ロジックの実装。

### フェーズ 3: 103イベントデータベースと日本国内マクロの拡張
- 総務省や内閣府のオープンAPI/CSVデータから、国内CPIや日銀短観、政策金利の取得経路を追加。
- イールドカーブの逆転、住宅着工件数、労働市場の変化など、すべてのマクロ指標（計103イベント）を網羅。

### フェーズ 4: プレミアムデスクトップGUI (PySide6)
- Streamlitに代わり、PySide6（Qt for Python）によるダークモード基調のデスクトップUIを構築。
- インタラクティブなグリッド表示と、リアルタイム検索・ソート対応のスクリーナーテーブルを実装。

### フェーズ 5: ローカルキャッシュとスタンドアロン配布
- SQLiteによる取得データのローカルキャッシュを実装し、オフライン動作とAPI呼び出し回数の最適化。
- PyInstallerを用い、macOS用（DMG/APP）およびWindows用（EXE）のスタンドアロンバイナリパッケージを生成。
