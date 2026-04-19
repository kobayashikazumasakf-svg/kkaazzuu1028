# AI薬局システムチェッカー システム設計書

**バージョン**: v1.0  
**作成日**: 2026年4月19日  

---

## 1. チェックルール仕様

### 1.1 ハイリスク薬リスト（特定薬剤管理指導加算 対象）

厚生労働省告示に基づく主要ハイリスク薬カテゴリ：

```
- 抗悪性腫瘍薬（経口）
- 免疫抑制薬
- 不整脈用薬
- 抗てんかん薬
- 血液凝固阻止薬（ワルファリン、DOACなど）
- ジギタリス製剤
- テオフィリン製剤
- カリウム製剤（注射）
- 精神神経用薬（リチウム製剤など）
- 糖尿病用薬（インスリン、SU薬など）
- 膵臓ホルモン剤
- 抗HIV薬
```

### 1.2 算定ルール定義（YAML形式）

```yaml
rules:
  - id: A-01
    name: 特定薬剤管理指導加算漏れ
    category: 加算漏れ
    priority: HIGH
    condition:
      - 処方薬にハイリスク薬が含まれる
      - かつ: 特定薬剤管理指導加算1 または 2 が算定されていない
    alert_message: "ハイリスク薬({drug_name})が処方されていますが特定薬剤管理指導加算が算定されていません"
    expected_points: 10  # 加算1の場合

  - id: A-02
    name: 吸入薬指導加算漏れ
    category: 加算漏れ
    priority: HIGH
    condition:
      - 処方薬に吸入薬が含まれる
      - かつ: 吸入薬指導加算が算定されていない
    alert_message: "吸入薬({drug_name})が処方されていますが吸入薬指導加算が算定されていません"
    expected_points: 30

  - id: B-01
    name: 薬剤料計算ミス
    category: 算定ミス
    priority: HIGH
    condition:
      - 算定薬剤料 ≠ Σ(薬価 × 数量 × 日数) / 10 （端数処理考慮）
      - 差異が1点以上
    alert_message: "薬剤料の計算に不一致があります（算定:{calculated}点 / 正値:{expected}点）"

  - id: B-04
    name: 向精神薬投与日数超過
    category: 算定ミス
    priority: CRITICAL
    condition:
      - 向精神薬（第1種・第2種）の投与日数が規定日数を超過
    alert_message: "向精神薬({drug_name})の投与日数({days}日)が上限を超えています"
```

---

## 2. データスキーマ

### 2.1 レセプトデータ（匿名化後）

```sql
CREATE TABLE receipts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    anon_patient_id VARCHAR(64) NOT NULL,  -- ハッシュ化患者ID
    receipt_yearmonth CHAR(6) NOT NULL,    -- YYYYMM
    pharmacy_id     VARCHAR(20) NOT NULL,
    total_points    INTEGER NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE receipt_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    receipt_id      UUID REFERENCES receipts(id),
    item_type       VARCHAR(20) NOT NULL,  -- '調剤技術料'|'薬剤料'|'加算'|'管理料'
    item_code       VARCHAR(20) NOT NULL,  -- 診療報酬コード
    item_name       VARCHAR(200) NOT NULL,
    points          INTEGER NOT NULL,
    quantity        DECIMAL(10,2),
    days            INTEGER
);

CREATE TABLE check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    receipt_id      UUID REFERENCES receipts(id),
    rule_id         VARCHAR(10) NOT NULL,
    rule_name       VARCHAR(200) NOT NULL,
    category        VARCHAR(20) NOT NULL,
    priority        VARCHAR(10) NOT NULL,
    alert_message   TEXT NOT NULL,
    expected_points INTEGER,
    pharmacist_verdict VARCHAR(10),  -- 'correct'|'false_positive'|'pending'
    pharmacist_note TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

### 2.2 診療報酬マスタ

```sql
CREATE TABLE drug_master (
    drug_code       VARCHAR(12) PRIMARY KEY,
    drug_name       VARCHAR(200) NOT NULL,
    drug_price      DECIMAL(10,2) NOT NULL,  -- 薬価（円）
    is_high_risk    BOOLEAN DEFAULT FALSE,
    high_risk_category VARCHAR(100),
    is_inhaler      BOOLEAN DEFAULT FALSE,
    is_psychotropic BOOLEAN DEFAULT FALSE,
    psychotropic_class VARCHAR(10),           -- '1種'|'2種'|'3種'
    max_dispensing_days INTEGER,
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE fee_master (
    fee_code        VARCHAR(12) PRIMARY KEY,
    fee_name        VARCHAR(200) NOT NULL,
    points          INTEGER NOT NULL,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    calc_conditions JSONB                     -- 算定条件
);
```

---

## 3. APIエンドポイント設計

### 3.1 レセプト取込み・チェック実行

```
POST /api/v1/receipts/upload
  Request: multipart/form-data (CSV or XML)
  Response: { job_id: string, receipt_count: number }

GET /api/v1/receipts/jobs/{job_id}
  Response: { status: 'processing'|'completed'|'failed', progress: number }

GET /api/v1/receipts/{yearmonth}/check-results
  Query: ?priority=HIGH&category=加算漏れ&verdict=pending
  Response: { results: CheckResult[], total: number, summary: Summary }
```

### 3.2 薬剤師フィードバック

```
PUT /api/v1/check-results/{result_id}/verdict
  Request: { verdict: 'correct'|'false_positive', note: string }
  Response: { updated: true }
```

### 3.3 レポート・統計

```
GET /api/v1/reports/monthly/{yearmonth}
  Response: {
    total_receipts: number,
    alerts_count: number,
    missed_additions_points: number,  -- 加算漏れ合計点数
    calculation_errors: number,
    false_positive_rate: number,
    top_issues: Issue[]
  }
```

---

## 4. チェックエンジン処理フロー

```python
# 処理フローの概要（擬似コード）

def process_receipt(receipt: Receipt) -> list[CheckResult]:
    results = []
    
    # 薬剤情報をマスタと照合
    drugs = enrich_with_master(receipt.prescribed_drugs)
    
    # カテゴリA: 加算漏れチェック
    results += check_high_risk_drug_addition(receipt, drugs)
    results += check_inhaler_addition(receipt, drugs)
    results += check_dispensing_guidance_category(receipt)
    results += check_duplication_prevention_addition(receipt)
    
    # カテゴリB: 算定ミスチェック
    results += check_drug_fee_calculation(receipt, drugs)
    results += check_monthly_limit(receipt)
    results += check_dispensing_days_limit(receipt, drugs)
    results += check_required_comments(receipt)
    
    # カテゴリC: 返戻リスクチェック
    results += check_diagnosis_drug_match(receipt, drugs)
    results += match_past_rejection_patterns(receipt)
    
    return sorted(results, key=lambda r: r.priority)
```

---

## 5. ダッシュボード画面仕様

### 5.1 メイン画面（アラート一覧）

```
┌─────────────────────────────────────────────────────┐
│  AI薬局システムチェッカー      2026年4月分          │
│  ケーファーマシー株式会社                           │
├─────────────────────────────────────────────────────┤
│  サマリー                                           │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐ │
│  │処理枚数  │ │加算漏れ  │ │算定ミス  │ │推定回収│ │
│  │ 1,000枚 │ │  87件   │ │  23件   │ │+6.8万円│ │
│  └──────────┘ └──────────┘ └──────────┘ └────────┘ │
├─────────────────────────────────────────────────────┤
│  アラート一覧  [未確認のみ ▼] [優先度：高 ▼]       │
│                                                     │
│  ● [高] レセプト#1042 - 特定薬剤管理指導加算漏れ   │
│    ワーファリン処方あり / 加算算定なし / +10点      │
│    [正しい] [誤検知] [詳細▼]                       │
│                                                     │
│  ● [高] レセプト#1078 - 吸入薬指導加算漏れ        │
│    フルティフォーム処方あり / 加算算定なし / +30点  │
│    [正しい] [誤検知] [詳細▼]                       │
│                                                     │
│  ▲ [中] レセプト#1103 - 薬剤料計算不一致          │
│    算定値:234点 / 正値:238点 / 差異:4点            │
│    [正しい] [誤検知] [詳細▼]                       │
└─────────────────────────────────────────────────────┘
```

### 5.2 月次レポート画面

- 加算漏れ種別グラフ（棒グラフ）
- 月次トレンド（折れ線グラフ）
- 薬剤師別確認率
- 診療報酬改定後の影響サマリー

---

## 6. 診療報酬改定対応プロセス

```
改定告示（厚生労働省）
        ↓
マスタDB更新（改定施行日の1週間前までに完了）
        ↓
ルールエンジン更新・テスト
        ↓
ユーザー通知（変更点のお知らせ）
        ↓
施行日から新ルールで運用開始
```

**改定スケジュール（通常）**
- 診療報酬改定: 偶数年4月（次回: 2028年4月）
- 薬価改定: 毎年4月（市場実勢価格調査: 9月）

---

**文書管理**  
作成: 2026年4月19日  
次回レビュー: Phase 1完了時
