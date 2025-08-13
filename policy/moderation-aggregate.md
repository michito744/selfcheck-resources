+# moderation-aggregate — 黄信号 × FC強制v7（合流ルール）
+
+[![Validate Yellow Config](https://github.com/michito744/selfcheck-resources/actions/workflows/validate-yellow.yml/badge.svg?branch=main)](https://github.com/michito744/selfcheck-resources/actions/workflows/validate-yellow.yml)

**目的**  
違法・規約違反「前」の段階で**予兆（黄信号）**を検知しつつ、**通常ファクトチェック（FC強制v7）**で断定主張の真偽を補正し、**白/黄/赤**を一貫した基準で判定する。

---

## 1. 参照ドキュメント
- 黄信号（予兆検知）  
  - [TRIGGERS_v1.md](./yellow-signal/TRIGGERS_v1.md)（A–Fトリガの定義）  
  - [WORKFLOW_v1.md](./yellow-signal/WORKFLOW_v1.md)（黄判定後の処理フロー）  
  - [yellow.config.schema.json](./yellow-signal/yellow.config.schema.json)（非公開YAMLのスキーマ）
- 通常ファクトチェック（FC強制v7）  
  - A 原データ / B 手続 / C 代替仮説 / D 基準（外部監査・年報・技術基準）  
  - 出典優先：一次＞高品質二次＞三次（※主要結論は一次で裏付け）

---

## 2. 入力 / 出力

**入力（最低）**  
- `text`: 投稿本文（必須）
- `ctx`: 付随情報（任意）  
  `reply_to_is_official`（D用）／`has_media_named_entity`（D補助）／`cluster_metrics`（A・B・E用） 等

**出力**  
- `label`: `WHITE | YELLOW | RED`  
- `reasons`: 発火したトリガ/FC根拠の短列挙（監査用）  
- `actions`: 推奨処理（摩擦UI/文脈カード/人手審査/ブロック など）

---

## 3. 用語と射程

**黄信号 A–F（要約）**  
A：文脈シグナル（対象語×モラル/感情語の共起増）  
B：断定隣接（上位拡散群に断定語が多発）  
C：行動語（日時・場所・集合/不買/デモ 等）  
D：ターゲティング補助（公式/事業アカ直撃、画像内の屋号・校章・個人名 等）  
E：連投率（同一アカ短期多投）  
F：保護属性接触（人種/国籍/宗教/性別/障害/年齢/深刻な疾病 への攻撃）

**FC強制v7（要約）**  
- 対象：**検証可能な主張**が含まれる投稿のみ起動  
- 結果：`True | False | Unverified`（一次優先の確度）

---

## 4. 判定フロー（合流ロジック・簡易）

1) **前処理（軽分類）**  
   - 主観（感想）か / 検証可能な主張か  
   - ターゲット種別（個人/団体/地名/属性）  
   - 断定語 / 行動語 / 保護属性語 の有無

2) **黄信号（A–F）**  
   - C・D・F：**本文/文脈だけで決まる**ので即評価  
   - A・B・E：`cluster_metrics` が無ければ **不明=0** で計算

3) **FC強制v7**（主張があるときのみ）  
   - 一次（公式原典）で照合 → `True/False/Unverified` を得る

4) **合流判定（最終色）**  
   - **F（属性攻撃）** → **RED**（規約直撃。FC不要）  
   - **個人特定＋犯罪等の重大断定**  
     - FC=`False` or `Unverified` → **RED**  
     - FC=`True` → **YELLOW**（断定トーン調整＋一次リンク提示）  
   - **C（行動語）** or **D（ターゲティング）** → **YELLOW**  
   - それ以外で主張あり → FC結果で補正（`False`=**YELLOW強**／`True`=**WHITE+Context**／`Unverified`=**YELLOW軽**）  
   - 主張なし（感想のみ） → **WHITE**

**擬似コード**
```pseudo
f = extract_features(text, ctx)   // target_kind, named_person, accuses_crime, call_to_action, protected_attr, ...
y = score_yellow(f, cluster_metrics?) // A–F（不明は0）

if f.protected_attr && f.calls_for_exclusion: return RED

if f.named_person && f.accuses_crime:
  fc = run_factcheck(text)
  return (fc in {False, Unverified}) ? RED : YELLOW

if f.call_to_action or f.targets_official: return YELLOW
if not f.has_assertion: return WHITE

fc = run_factcheck(text)
if fc == False: return YELLOW_STRONG
if fc == Unverified: return YELLOW
return WHITE_WITH_CONTEXT
