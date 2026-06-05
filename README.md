# Microsoft Fabric Data Agent ハンズオン
## 〜 旅行販売データで始める AI データ分析 〜

**所要時間:** 約 60 分  
**難易度:** 初級〜中級  
**対象者:** Microsoft Fabric を初めて触る方、Data Agent に興味がある方

---

## 概要

本ハンズオンでは、旅行代理店の販売データと顧客レビューデータを活用し、Microsoft Fabric の **Data Agent** を作成します。Data Agent とは、自然言語で質問するだけでデータを分析・回答してくれる AI エージェントです。

### 使用データ

| ファイル | 内容 | 主なカラム |
|---|---|---|
| `Sales.csv` | 旅行販売トランザクション | 旅行先、カテゴリ（国内/海外）、日程、価格、人数、年代 |
| `Sales_review.csv` | 顧客レビュー | 評価（1〜5）、感情（ポジティブ/ネガティブ）、コメント |

### ゴール

ハンズオン終了後、以下のような質問を自然言語で Data Agent に投げかけ、回答を得られるようになります。

- 「今年の売上を月別に教えてください」
- 「最も人気の旅行先はどこですか？」
- 「30代に人気の海外旅行先を教えてください」
- 「評価が低い旅行先はどこですか？」
- 「ハワイの平均評価と売上を教えてください」

---

## アジェンダ

| # | ステップ | 所要時間 |
|---|---|---|
| 0 | 事前準備・環境確認 | 5 分 |
| 1 | Lakehouse の作成とデータのアップロード | 15 分 |
| 2 | セマンティックモデルの作成 | 10 分 |
| 3 | Data Agent の作成と設定 | 15 分 |
| 4 | Data Agent への質問とテスト | 10 分 |
| 5 | まとめと応用 | 5 分 |

---

## 事前準備

### 必要な環境

- Microsoft Fabric へのアクセス権限（Contributor 以上推奨）
- **有償の Fabric 容量が割り当てられたワークスペース（F2 以上）**
- 本ハンズオンで使用するファイル
  - `data/Sales.csv`
  - `data/Sales_review.csv`

> ⚠️ **ライセンスに関する注意:**  
> Data Agent は **有償の Fabric 容量（F SKU）** が割り当てられたワークスペースでのみ利用可能です。  
> Microsoft Fabric Trial（60日間試用）や Power BI Premium Per User（PPU）では Data Agent を利用できません。  
> 事前にワークスペースに適切な Fabric 容量が割り当てられているか管理者に確認してください。

### Fabric へのアクセス確認

1. ブラウザで [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com) を開く
2. 組織アカウントでサインイン
3. ホーム画面が表示されることを確認

---

## Step 0: ワークスペースの確認・作成（5 分）

### ワークスペースへの移動

1. 左側ナビゲーションで「**ワークスペース**」を選択
2. 使用するワークスペースをクリックして開く
   - ワークスペースがない場合は「**+ 新しいワークスペース**」で作成

---

## Step 1: Lakehouse の作成とデータのアップロード（15 分）

### 1-1. Lakehouse の作成

1. ワークスペース画面で「**+ 新しいアイテム**」をクリック
2. 「**Lakehouse**」を選択
3. 名前に `TravelSalesLakehouse` と入力して「**作成**」をクリック

   > **ヒント:** 名前はわかりやすければ何でも構いません

4. Lakehouse のホーム画面が表示されることを確認

---

### 1-2. CSV ファイルのアップロード

1. Lakehouse 画面の左パネルで「**Files**」を右クリック
2. 「**新しいサブフォルダー**」をクリックし、フォルダ名を `raw` と入力
3. 作成した `raw` フォルダを右クリック →「**ファイルのアップロード**」を選択
4. 以下の 2 ファイルを選択してアップロード
   - `Sales.csv`
   - `Sales_review.csv`

5. アップロード完了後、`raw` フォルダ内に 2 つのファイルが表示されることを確認

---

### 1-3. CSV から Delta テーブルへの変換

Data Agent が SQL でデータを検索できるよう、CSV を **Delta テーブル**に変換します。

1. `Sales.csv` を右クリック →「**テーブルの読み込み**」を選択
2. 以下の設定を入力
   - テーブル名: `sales`
   - ヘッダー行を使用: **オン**
   - 区切り文字: **カンマ (,)**

3. 「**テーブルの読み込み**」をクリック

4. 同様に `Sales_review.csv` を右クリック →「**テーブルの読み込み**」を選択
   - テーブル名: `sales_review`
   - ヘッダー行を使用: **オン**
   - 区切り文字: **カンマ (,)**

5. 両方のテーブルが左パネルの「**Tables**」に表示されることを確認

   > **参考 — テーブルのカラム構成:**  
   > `sales`: Transaction_ID / Date / Travel_destination / Category / Schedule / Price / Price_per_person / Number_of_people / Age_group  
   > `sales_review`: Transaction_ID / Travel_destination / Rating / Emotions / Comments  
   > ※ 詳細は付録のカラム一覧を参照してください。

---

## Step 2: セマンティックモデルの作成（10 分）

セマンティックモデルとは、テーブル間のリレーション・メジャー・列の設定などを定義した Power BI 用のデータモデルです。Data Agent のデータソースとして接続することで、テーブルの結合関係が自動的に考慮され、より正確な分析が可能になります。

---

### 2-1. Lakehouse からセマンティックモデルを作成

1. `TravelSalesLakehouse` を開く
2. リボン上部の「**新しいセマンティックモデル**」をクリック
3. 名前に `TravelSalesModel` と入力
4. テーブル一覧から以下の 2 つにチェックを入れ、「**確認**」をクリック
   - ✅ `sales`
   - ✅ `sales_review`

5. セマンティックモデルが作成され、モデル編集画面が開くことを確認

---

### 2-2. テーブルのリレーションを設定

2 つのテーブルを `Transaction_ID` で結合するリレーションを定義します。

1. モデル編集画面で `sales` テーブルと `sales_review` テーブルが表示されていることを確認
2. `sales` テーブルの `Transaction_ID` 列を `sales_review` テーブルの `Transaction_ID` 列へ**ドラッグ＆ドロップ**
3. リレーション設定ダイアログで以下を確認して「**OK**」をクリック
   - カーディナリティ: **一対多 (1:*)** （`sales` が「1」側、`sales_review` が「多」側）
   - クロスフィルタの方向: **単一**

4. 2 つのテーブルが線でつながれることを確認

---

> **ポイント:** セマンティックモデルにリレーションを定義しておくことで、Data Agent が自動的にテーブルを結合して回答を生成できるようになります。セマンティックモデルの変更は**自動保存**されます。

---

## Step 3: Data Agent の作成と設定（15 分）

### 3-1. Data Agent の作成

1. ワークスペース画面に戻る（左上のワークスペース名をクリック）
2. 「**+ 新しいアイテム**」をクリック
3. 検索バーに「**Data agent**」と入力し、「**Data agent**」を選択
4. 名前に `TravelSalesAgent` と入力して「**作成**」をクリック

---

### 3-2. データソースの追加

Data Agent に分析対象のデータを教えます。Step 2 で作成したセマンティックモデルをデータソースとして接続します。

1. Data Agent の画面が開いたら、「**データの追加**」をクリック
2. ドロップダウンから「**データ ソース**」を選択
3. `TravelSalesModel` を検索して選択（種類: セマンティックモデル）
4. 「**接続**」をクリック

---

### 3-3. テーブルの選択

1. データソース追加後、テーブル一覧が表示される
2. 以下の 2 つのテーブルにチェックを入れる
   - ✅ `sales`
   - ✅ `sales_review`
3. 「**確認**」または「**追加**」をクリック

> **ポイント:** セマンティックモデルを使うことで、Step 2 で設定したリレーションが Data Agent にも引き継がれます。

---

### 3-4. エージェントの指示の設定

Agent の振る舞いとデータの内容を定義する指示を設定します。テーブルやカラムの説明もここに含めることで、Data Agent の回答精度が向上します。

1. 画面上部の「**セットアップ**」タブを選択
2. 「**エージェントの指示**」欄に以下のテキストをコピー＆ペースト:

```
あなたは旅行代理店の売上分析を行うデータアナリストです。
salesテーブルとsales_reviewテーブルを使って、ユーザーの質問に日本語で丁寧に答えてください。

## テーブル情報

### salesテーブル
旅行代理店の販売トランザクションデータ。各行が1件の旅行予約を表す。2024年1月〜3月の取引データが含まれる。
- Transaction_ID: 取引を一意に識別するID（例: S00001）
- Date: 取引日（YYYY/MM/DD形式）
- Travel_destination: 旅行の目的地（例: 京都、ハワイ、パリ）
- Category: 国内旅行か海外旅行かの分類（値: 国内, 海外）
- Schedule: 旅行の日程（例: 2泊3日, 4泊5日）
- Price: 旅行合計価格（円）
- Price_per_person: 一人あたりの旅行価格（円）
- Number_of_people: 旅行参加人数
- Age_group: 代表者の年代（例: 20代, 30代, 40代, 50代, 60代）

### sales_reviewテーブル
旅行後の顧客レビューデータ。salesテーブルのTransaction_IDで結合可能。各行が1件の旅行に対するレビューを表す。
- Transaction_ID: 取引ID（salesテーブルと結合するキー）
- Travel_destination: レビュー対象の旅行先
- Rating: 顧客満足度（1〜5の整数、5が最高評価）
- Emotions: レビューの感情分析結果（値: ポジティブ, ネガティブ）
- Comments: 顧客の自由記述コメント

## 回答ルール
- 金額は「円」単位で表示し、3桁区切りのカンマを使用してください
- 旅行先名はデータそのままの表記を使用してください
- 数値の比較や集計が必要な場合は、SQLを使って正確に計算してください
- 回答は簡潔にまとめ、必要に応じて表形式で見やすく表示してください
- salesとsales_reviewはTransaction_IDで結合できます
- Ratingの平均値は小数第2位まで表示してください（例: 4.13）。整数や小数第1位に丸めないでください
```

3. 「**保存**」をクリック

---

## Step 4: Data Agent への質問とテスト（10 分）

### 4-1. 基本的な質問

画面下部のチャット入力欄に質問を入力して、Data Agent の回答を確認しましょう。

#### 質問 1: カテゴリ別集計

```
国内旅行と海外旅行の件数と売上合計を教えてください。
```

**期待される回答の例:**

| カテゴリ | 件数 | 売上合計 |
|---|---|---|
| 国内 | XX | X,XXX,XXX 円 |
| 海外 | XX | X,XXX,XXX 円 |

---

#### 質問 2: 人気旅行先

```
旅行件数が多い旅行先トップ5を教えてください。
```

**期待される回答の例:**

| 旅行先 | 件数 |
|---|---|
| ハワイ | X |
| 大阪 | X |
| 九州 | X |
| ... | ... |

---

### 4-2. 顧客・レビュー分析の質問

#### 質問 3: 年代別分析

```
30代に人気の旅行先を上位5件教えてください。
```

---

#### 質問 4: レビュー分析

```
旅行先ごとの評価（Rating）の平均を小数第2位まで計算し、最も高い旅行先と最も低い旅行先を教えてください。
```

---

#### 質問 5: テーブル結合を使った分析

```
ハワイへの旅行について、売上合計・平均価格・平均評価を教えてください。
```

**期待される回答の例:**
> ハワイへの旅行データ:
> - 件数: X 件
> - 売上合計: X,XXX,XXX 円
> - 平均価格（一人あたり）: XXX,XXX 円
> - 平均評価: X.X / 5.0

---

### 4-3. DAX クエリの確認

Data Agent が生成した DAX クエリを確認してみましょう。

> **補足:** データソースにセマンティックモデルを使用している場合、Data Agent は SQL ではなく **DAX（Data Analysis Expressions）** を生成します。

1. 回答の下にある「**クエリを確認**」または「**DAX を表示**」をクリック
2. 自動生成された DAX クエリを確認する

**ポイント:** Data Agent が正確な DAX を生成しているか確認することで、回答の信頼性を検証できます。

---

## Step 5: まとめと応用（5 分）

### ハンズオンのまとめ

本ハンズオンでは以下を実施しました:

1. ✅ **Lakehouse の作成** - CSV データを Delta テーブルとして保存
2. ✅ **セマンティックモデルの作成** - テーブル間のリレーションを定義
3. ✅ **Data Agent の作成** - セマンティックモデルをデータソースとして接続
4. ✅ **メタデータの設定** - テーブルとカラムに説明を追加して精度向上
5. ✅ **自然言語での分析** - チャットで多様なビジネス質問に回答

---

## 補足: Data Agent の評価（Evaluation）

> 📖 公式ドキュメント: [データ エージェントを評価する - Microsoft Fabric | Microsoft Learn](https://learn.microsoft.com/ja-jp/fabric/data-science/evaluate-data-agent)

`evaluation/` フォルダには、Data Agent の回答精度をプログラムで評価するためのファイルが含まれています。

### ファイル構成

| ファイル | 説明 |
|---|---|
| `evaluation_questions.csv` | 質問と期待される回答（グランドトゥルース） |
| `evaluate_data_agent.ipynb` | 評価を実行する Fabric Notebook |

### 評価対象の質問

Step 4 で実施した以下の 5 問を評価します。

| # | 質問 |
|---|---|
| 1 | 国内旅行と海外旅行の件数と売上合計を教えてください。 |
| 2 | 旅行件数が多い旅行先トップ5を教えてください。 |
| 3 | 30代に人気の旅行先を上位5件教えてください。 |
| 4 | 旅行先ごとの評価（Rating）の平均を小数第2位まで計算し、最も高い旅行先と最も低い旅行先を教えてください。 |
| 5 | ハワイへの旅行について、売上合計・平均価格・平均評価を教えてください。 |

### 手順

#### 1. CSV ファイルを OneLake にアップロード

1. Fabric の **TravelSalesLakehouse** を開く
2. 左のエクスプローラーで **Files** を展開し、`data` フォルダを開く  
   （存在しない場合は `data` フォルダを新規作成）
3. **「アップロード」** をクリックし、`evaluation/evaluation_questions.csv` をアップロード

> アップロード先のパス: `Files/data/evaluation_questions.csv`

#### 2. Notebook を Fabric にインポート

1. Fabric ワークスペースの **「新しいアイテム」→「ノートブックのインポート」** をクリック
2. `evaluation/evaluate_data_agent.ipynb` を選択してインポート
3. インポートされた Notebook を開く

#### 3. Lakehouse をアタッチ

1. Notebook 右側の **「Lakehouse を追加」** をクリック
2. **TravelSalesLakehouse** を選択してアタッチ

#### 4. Notebook を実行

セルを上から順に実行します。

| ステップ | 内容 |
|---|---|
| Step 1 | `fabric-data-agent-sdk` のインストール |
| Step 2 | CSV ファイルの読み込み |
| Step 3 | 評価の実行（全質問を Data Agent に送信） |
| Step 4 | 評価サマリーの表示（合否・スコア） |
| Step 5 | 評価詳細の表示（各質問の回答と期待値の比較） |

> **注意:** Step 3 の実行には数分かかります。Data Agent が全質問に回答するまでお待ちください。

#### 5. 結果の確認

- **Step 4** でスコアの全体像（正答率など）が表示されます
- **Step 5** で各質問ごとの回答内容と期待値を詳細比較できます

### Data Agent の精度を高めるコツ

| コツ | 内容 |
|---|---|
| テーブルの説明を充実させる | どんなデータが入っているか、業務文脈を日本語で記述する |
| カラムの説明を追加する | 特に略語や業務用語はわかりやすく説明する |
| サンプル質問を登録する | よく使う質問とその期待回答を登録するとより精度が上がる |
| エージェントの指示を調整する | 回答フォーマットや使用言語を明示する |

### 応用アイデア

- **Power BI との連携:** Data Agent を Power BI レポートに埋め込む
- **データの追加:** 四半期データや顧客マスタを追加してより深い分析を実現
- **他チームへの共有:** ワークスペース権限を設定して Data Agent を組織で共有

---

## トラブルシューティング

### Q: テーブルが Data Agent に表示されない

**A:** Lakehouse の SQL 分析エンドポイントにデータが反映されるまで数分かかることがあります。しばらく待ってから再試行してください。

---

### Q: 日本語の質問がうまく認識されない

**A:** エージェントの指示に「必ず日本語で回答してください」と明示的に追加してください。また、カラムの説明を日本語で充実させると精度が向上します。

---

### Q: Data Agent が間違った回答をする

**A:** 以下を確認してください:
1. テーブルとカラムの説明が正確か確認する
2. SQL を表示して、生成されたクエリが正しいか確認する
3. 質問をより具体的に言い換える（例: 「売上」→「Priceカラムの合計」）

---

### Q: 「Data agent」が新しいアイテムの一覧に表示されない

**A:** Data Agent 機能は Fabric の Preview 機能として提供されています。ワークスペース設定で「**プレビュー機能を有効化**」されているか確認してください。

---

## 参考リンク

- [Microsoft Fabric Data Agent 公式ドキュメント](https://learn.microsoft.com/ja-jp/fabric/data-science/agent)
- [データ エージェントを構成するためのベスト プラクティス](https://learn.microsoft.com/ja-jp/fabric/data-science/data-agent-configuration-best-practices)
- [Fabric Lakehouse とは](https://learn.microsoft.com/ja-jp/fabric/data-engineering/lakehouse-overview)
- [Delta Lake の概要](https://learn.microsoft.com/ja-jp/fabric/data-engineering/lakehouse-and-delta-tables)

---

## 付録: テーブルのカラム一覧

### `sales` テーブル

| カラム名 | 説明 | 例 |
|---|---|---|
| Transaction_ID | 取引を一意に識別する ID | S00001 |
| Date | 取引日（YYYY/MM/DD） | 2024/01/24 |
| Travel_destination | 旅行先 | 京都 |
| Category | 国内 / 海外 の分類 | 国内 |
| Schedule | 旅行の日程 | 2泊3日 |
| Price | 旅行合計価格（円） | 64000 |
| Price_per_person | 一人あたり価格（円） | 32000 |
| Number_of_people | 旅行参加人数 | 2 |
| Age_group | 代表者の年代 | 20代 |

### `sales_review` テーブル

| カラム名 | 説明 | 例 |
|---|---|---|
| Transaction_ID | 取引 ID（sales と結合するキー） | S00001 |
| Travel_destination | レビュー対象の旅行先 | 京都 |
| Rating | 顧客満足度（1〜5、5が最高） | 3 |
| Emotions | 感情分析結果（ポジティブ / ネガティブ） | ポジティブ |
| Comments | 顧客の自由記述コメント | 寺社仏閣が素晴らしかった |

---

## 付録: 参考 SQL クエリ集

Lakehouse の SQL 分析エンドポイントで、Step 4 の各質問に対応する結果を事前確認できるクエリです。

### 質問 1: 国内・海外の件数と売上合計

```sql
SELECT
    Category,
    COUNT(*) AS trips,
    SUM(Price) AS total_sales
FROM sales
GROUP BY Category
ORDER BY Category;
```

---

### 質問 2: 旅行件数が多い旅行先トップ 5

```sql
SELECT TOP 5
    Travel_destination,
    COUNT(*) AS trips
FROM sales
GROUP BY Travel_destination
ORDER BY trips DESC;
```

---

### 質問 3: 30 代に人気の旅行先上位 5 件

```sql
SELECT TOP 5
    Travel_destination,
    COUNT(*) AS trips
FROM sales
WHERE Age_group = '30代'
GROUP BY Travel_destination
ORDER BY trips DESC;
```

---

### 質問 4: 旅行先別の平均評価（小数第 2 位）

```sql
SELECT
    Travel_destination,
    ROUND(AVG(CAST(Rating AS FLOAT)), 2) AS avg_rating
FROM sales_review
GROUP BY Travel_destination
ORDER BY avg_rating DESC;
```

---

### 質問 5: ハワイの売上合計・平均価格・平均評価

```sql
SELECT
    COUNT(*)                                        AS trips,
    SUM(s.Price)                                    AS total_sales,
    ROUND(AVG(CAST(s.Price_per_person AS FLOAT)), 0) AS avg_price_per_person,
    ROUND(AVG(CAST(r.Rating AS FLOAT)), 2)          AS avg_rating
FROM sales s
JOIN sales_review r ON s.Transaction_ID = r.Transaction_ID
WHERE s.Travel_destination = 'ハワイ';
```

---

*Microsoft Fabric Data Agent ハンズオン — 作成日: 2026年*
