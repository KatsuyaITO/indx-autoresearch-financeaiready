---
name: deepen-whitepaper
description: >
  AI-Ready Finance ホワイトペーパー（whitepaper/）を最新情報で分厚くするための研究手順。
  各章を並列エージェントで Web リサーチ → GitHub Issues 化 → 章の加筆 + 引用追加 → 統合 →
  コンパイル検証まで実行する。ホワイトペーパーの加筆・更新・章の拡充を頼まれたときに使う。
---

# スキル: ホワイトペーパーを最新情報で分厚くする（deepen-whitepaper）

このスキルは、`whitepaper/` の LaTeX ホワイトペーパーを **「各章ごとに並列エージェントを起動し、
Web リサーチ → GitHub Issues 化 → 章の加筆 → 引用の追加 → 統合 → コンパイル検証」** の順で
分厚くするための、再現可能な手順書である。2026-07 の拡張パス（Issues 295→359、参照 215→289、
本文 1,817→2,271 行）で実際に用いた流れをそのまま定式化している。

> **大原則（`whitepaper/CLAUDE.md` と一致させること）**
> 1. **GitHub Issues がグラウンドトゥルース。** Web で得た新情報は必ず Issue 化してから本文に書く。
> 2. **並列作業。** section ごとにエージェントを起動し、できるだけ同時に走らせる。
> 3. **読者はプロだが、分かりやすさ・図表を忘れない。**
> 4. **鳥の目と虫の目。** 全体像の概観 ＋ 論文を読まなくてよいレベルの具体解説。
> 5. **全てに引用を。** `references.bib` を更新し、あらゆる主張に `\cite{}` を付ける。捏造は厳禁。

---

## 全体フロー（一目で）

```
STEP 0  下調べ（章構成・Issue 規約・ラベル・参照数・コンパイル環境の把握）
STEP 1  refs/ ディレクトリを用意（並列書き込み衝突を避けるため）
STEP 2  ★章ごとに並列エージェントを起動★
          各エージェント: Web調査 → Issue作成 → 自分の章.tex を加筆 → refs/refs-NN.bib に新規参照
STEP 3  全 refs/refs-NN.bib を references.bib へマージ（キー衝突チェック）
STEP 4  統合パス（本人が担当）: 数値・日付・相互参照の整合、要約系章の更新
STEP 5  コンパイル検証（lualatex → biber → lualatex ×2）＋ エラーゼロを確認
```

**依存関係のコツ:** 本論の章（01/02/03/04/05/07）は互いに独立なので**並列**でよい。
要約・統合系の章（00 エグゼクティブサマリー / 06 横断考察 / 08 結論）は他章の成果に依存するので、
**並列エージェントの完了後に自分でまとめて更新**する（＝ウェーブ2）。

---

## 前提として押さえる事実（STEP 0 で確認）

- **章ファイル:** `whitepaper/sections/00〜09-*.tex`。`whitepaper.tex` が `\input` で束ねる。
  - 00 エグゼクティブサマリー / 01 導入 / 02 データ基盤 / 03 AI-Nativeファンド /
    04 投資エージェント / 05 LLM分析 / 06 横断考察 / 07 未解決ギャップ / 08 結論 / 09 参照。
- **参照:** `whitepaper/references.bib`（biblatex + biber、`\addbibresource{references.bib}` は1本のみ）。
- **Issues:** リポジトリの GitHub Issues が一次情報の台帳。既存の命名・ラベル規約に必ず合わせる。
- **コンパイル:** `lualatex`（LuaLaTeX / ltjsarticle・日本語）+ `biber`。プリアンブルに
  `fnode / fnode blue / fnode accent / farr / casestudy / insight` などの共通スタイルが定義済み。
  **図・囲みはこれらを再利用し、プリアンブルは再定義しない。**

STEP 0 の実行例:

```bash
cd whitepaper
gh auth status && git remote -v
gh issue list --limit 5 --state all           # 命名・ラベルの実例を見る
gh label list --limit 100                      # 既存ラベルを把握（再利用が原則）
gh issue view <番号>                            # 本文フォーマット（Source/Link/末尾注記）を確認
grep -c '^@' references.bib                     # 既存参照数（拡張前の基準値）
wc -l sections/*.tex                            # 各章の分量（拡張前の基準値）
which lualatex biber                            # ツールチェーン確認
```

---

## STEP 1 — refs/ を用意

複数エージェントが同じ `references.bib` に同時書き込みするとファイルが壊れる。そこで
**各章は自分専用の断片ファイル `refs/refs-NN.bib` にだけ新規参照を書き**、最後に本人がマージする。

```bash
mkdir -p whitepaper/refs
```

---

## STEP 2 — 章ごとに並列エージェントを起動（ウェーブ1）

`Agent` ツール（`general-purpose`）を **1メッセージ内で複数同時に**起動する。担当は本論の各章:
**01 / 02 / 03 / 04 / 05 / 07**。以下のテンプレートの `{{...}}` を章ごとに差し替える。

### エージェント用プロンプトテンプレート

```
あなたは日本語 LaTeX 研究ホワイトペーパー「AI-Ready Finance」の1章を拡張する。
作業ディレクトリ: <repo>/whitepaper

担当章: sections/{{NN-ファイル名}}.tex （{{章の主題}}）

STEP 0 — 先に読む:
- whitepaper/CLAUDE.md（執筆ルール） / 担当章の .tex（現状と LaTeX スタイル） /
  sections/00-executive-summary.tex（4本柱の枠組み把握）

執筆ルール（厳守）:
1. GitHub Issues がグラウンドトゥルース。Web で得た新事実は必ず Issue 化（STEP 2）。
2. 読者はプロ。ただし分かりやすさ・図表を重視。
3. 鳥の目（概観）＋虫の目（具体を論文なしで理解できるレベルで解説）。
4. すべての主張に \cite{}＋参照エントリ（STEP 3）。
5. 論文・著者・日付・数値・URL を絶対に捏造しない。必ず取得して裏取り。不確かなら
   数値を書かず研究の種類を記述する。

STEP 1 — Web リサーチ（多めに。2024〜2026 を重視）:
{{この章で深掘りすべき具体トピック・企業名・ベンチマーク名を列挙}}
WebSearch / WebFetch で実際に一次情報を開き、正確な数値・日付・組織名・URL を控える。堅い出典を15件以上。

STEP 2 — GitHub Issues 作成（新発見1件につき1 Issue、目安10〜18件）:
`gh issue create` を使い、既存規約に合わせる。
- タイトル接頭辞は性質で選ぶ:「Foundations:」（不変の事実）/「Latest:」（最近の動向）/「Takes:」（解釈・仮説）。
- ラベル: 必ず `auto-research,domain:finance` ＋ `related-work`/`research-news`/`hypothesis` のいずれか1つ
  ＋ 主題ラベル1〜3個。`gh label list --limit 100` で既存を再利用。合うものが無い時だけ
  `gh label create "<名前>"` で新設。
- 本文: 要約数文 ＋ 行 `**Source:** <媒体> · <日付>` ＋ `**Link:** <URL>` ＋ Web検証済みの注記。
- 既存 Issue の重複を作らない。作成前に `gh issue list --search "<キーワード>" --state all` で確認。

STEP 3 — 担当章の .tex を分厚くする:
新素材で章を大幅加筆。casestudy（検証済み具体例）と insight（そこから導く示唆・仮説）を対で置く。
比較表（booktabs/tabularx）や tikz 図を最低1つ、既存の fnode/farr スタイルを再利用して追加（プリアンブルは再定義しない）。
日本語アカデミックの語調を維持。新しい主張には必ず \cite{}。
編集してよいのは担当章ファイルのみ。references.bib・whitepaper.tex・他章は触らない。

STEP 4 — 参照断片:
新規参照はすべて refs/refs-{{NN}}.bib に書く。
- 追加前に references.bib を grep し、既に在る出典は既存キーを再利用（重複させない）。
- 新規キーは必ず接頭辞 `x{{NN}}` を付ける（例 x02foo2026）＝全体でのキー衝突を防ぐ。
- @online/@misc/@article に title/author(or org)/year/url/urldate を揃える。実在・取得済みの出典のみ。

自律的に進め、完了時に (a) 作成 Issue 番号一覧, (b) 追加した bib キー, (c) 章拡張の3行要約 を返す。確認は求めない。
```

**章ごとの STEP 1 差し替え例（要点のみ）:**
- **01 導入:** 市場規模・採用率統計（McKinsey/BCG/Gartner/Bloomberg Intelligence）、AI-ready vs AI-native の定義、MCP/エージェント台頭の年表、2023→2026 の変曲点。
- **02 データ基盤:** 金融文書パース（PDF/入れ子表/10-K）、RAG（階層/グラフ/エージェント型）、埋め込み・ベクトル検索、ナレッジグラフ、MCP 提供ベンダー（Bloomberg/LSEG/FactSet/Daloopa/S&P/Moody's）、ベンチマーク（FinanceBench/Fin-RATE/FinQA/DocFinQA 等）。
- **03 AI-Native ファンド:** Bridgewater/Renaissance/Two Sigma/Citadel/D.E.Shaw/Point72/QRT/WorldQuant/AQR/Balyasny/Millennium ＋ 銀行・運用会社（JPMorgan/Goldman/BlackRock/Morgan Stanley）＋ 新規参入（Lumenai/Magnetar）。**運用成績の数値は必ず出典明示、無ければ定性的に。**
- **04 投資エージェント:** マルチエージェント枠組み（TradingAgents/AlphaAgents/FINSABER/DeepFund/FinCon/MarketSenseAI/FinRobot 等）、DD・アイデア創出・監視・リスク、オーケストレーション類型、評価の厳密さ。
- **05 LLM 分析:** 抽出/要約/分類/推論/比較のベンチマークと限界、ハルシネーション・忠実性、Look-Ahead バイアス、ドメイン LLM（BloombergGPT/FinBERT/Fin-R1/FinGPT/Palmyra-Fin）、記号検証ハイブリッド。
- **07 未解決ギャップ:** 評価・再現性、忠実性保証、因果 vs 相関、データリーク、説明可能性・監査、モデルリスク統治、エージェント安全性、AI モノカルチャーのシステミックリスク、標準化・プライバシー・コスト。

---

## STEP 3 — 参照断片を references.bib にマージ

全エージェント完了後、本人が実行。**先に衝突を検査**してから追記する。

```bash
cd whitepaper
# (1) 断片内・断片間の重複キー、および既存 references.bib とのキー衝突を検査（どれも空であるべき）
grep -hoE '^@[a-z]+\{[^,]+' refs/refs-0*.bib | sed -E 's/^@[a-z]+\{//' | sort > /tmp/newkeys.txt
sort /tmp/newkeys.txt | uniq -d                                   # 断片間の重複
grep -oE '^@[a-z]+\{[^,]+' references.bib | sed -E 's/^@[a-z]+\{//' | sort > /tmp/oldkeys.txt
comm -12 /tmp/newkeys.txt /tmp/oldkeys.txt                        # 既存との衝突

# (2) バックアップして追記マージ
cp references.bib references.bib.bak
for n in 01 02 03 04 05 07; do
  printf '\n%% ---- section %s ----\n' "$n" >> references.bib
  cat "refs/refs-$n.bib" >> references.bib
done
grep -c '^@' references.bib                                       # 期待値 = 旧数 + 新規数
```

---

## STEP 4 — 統合パス（要約・整合。本人が担当）

並列エージェントは自章しか見ないので、**章をまたぐ整合は必ず本人が取る**。

- **数値の統一:** 総ソース件数（例「290件を超える」）、参照数などが本文・表紙・結論でずれていないか。
  拡張後の Issue 数（`gh issue list --state all --limit 400 | wc -l`）に合わせて更新。
- **日付・事実の整合:** ある章が既存の事実を更新した場合（例: EU AI Act の高リスク期限が
  Digital Omnibus で 2026-08-02→2027-12-02 に延期）、それを引用する**全章と図（tikz のノード・
  キャプション・軸ラベル）**を必ず反映させる。矛盾を残さない。
- **要約系章の更新:** 00（主要な発見の追記）・06（新しい横断示唆の織り込み）・08（総括・件数）を
  新素材に合わせて加筆。ここでは新キーも本文から `\cite{}` してよい（既にマージ済みなので解決する）。

---

## STEP 5 — コンパイル検証

```bash
cd whitepaper
lualatex -interaction=nonstopmode whitepaper.tex >/tmp/l1.log 2>&1
biber whitepaper                                    >/tmp/b.log  2>&1
lualatex -interaction=nonstopmode whitepaper.tex >/tmp/l2.log 2>&1
lualatex -interaction=nonstopmode whitepaper.tex >/tmp/l3.log 2>&1

grep -n '^! ' /tmp/l3.log                                        # TeX エラー（空であるべき）
grep -icE 'Citation .* undefined|Reference .* undefined' /tmp/l3.log   # 0 であるべき
grep 'Output written' /tmp/l3.log                                # ページ数を確認
# 引用キーが本当に解決したか（bbl 突き合わせ）:
for k in $(grep -hoE 'x0[0-9][a-zA-Z0-9_]+' sections/*.tex | sort -u); do
  grep -q "{$k}" whitepaper.bbl || echo "MISSING in bbl: $k"; done
```

初回 `lualatex` が exit 1 でも、`biber` 後の再ランで消えるなら問題ない（`.bbl` 再生成前の残骸）。
最終ランで **TeX エラー0・未定義引用0** を確認できれば完了。

---

## 落とし穴（過去に踏んだもの）

- **`references.bib` の未エスケープ特殊文字。** bib の title/note 内の `$ & _ # % ^ ~` は要エスケープ
  （`\$ \& \_ \# \% \^{} \~{}`）。特に `$` 単独は数式モードを開いて
  `Missing $ inserted` を起こす。マージ前に断片を
  `grep -nE '[^\\][#%^~$&]' refs/refs-0*.bib`（url 行を除く）で点検。
- **キー衝突。** 新規キーは必ず `xNN` 接頭辞。マージ前に STEP 3 の衝突検査を必ず通す。
- **並列書き込み。** エージェントに `references.bib`・`whitepaper.tex`・他章を触らせない（断片＋自章のみ）。
- **`\S` と CJK の連結。** `\S後述` のように制御綴と日本語が地続きだと未定義制御綴になる。`\S\ref{...}` か、参照でなければ `\S ` の後を空ける／「後述」等の素の語にする。
- **プリアンブル再定義の禁止。** 図・囲みは既存スタイルを使う。新 `\tikzset`/`\newtcolorbox` を章内で足さない。
- **`Kanji font shape ... undefined` は無害。** 明朝の斜体シェイプ代替の情報メッセージで、元から出る。エラーではない。
- **数値の捏造禁止。** 特に運用成績・市場規模・普及率は出典が取れなければ書かない。取れても
  「一次資料に遡れない参考値」は本文でその旨を明示する（既存本文の作法に倣う）。

---

## 完了チェックリスト

- [ ] 新発見はすべて Issue 化した（規約準拠・重複なし）。
- [ ] 担当章がすべて加筆され、casestudy/insight・表・図が追加された。
- [ ] 追加主張に `\cite{}` が付き、`refs/refs-NN.bib`→`references.bib` にマージ済み。
- [ ] 章をまたぐ数値・日付・相互参照・図が整合している。
- [ ] 00/06/08 の要約系章が新素材を反映している。
- [ ] 最終コンパイルが TeX エラー0・未定義引用0 で通り、`whitepaper.pdf` が生成された。
