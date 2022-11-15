# BigQuery (SQL) もくもく会
## スコープ
- SQLをとりあえず触ってみる
- DWH製品に触れてみる 
## スコープ外
- DWHとは（業務DBと何が違うの？）
- 運用におけるナレッジ
- DDL系の話　などなど

## SQLに慣れよう
### SQLて何よ？
- (一言で)データを操作や定義することに特化した言語
- 宣言型言語（対義語：手続型言語）
- SQLはDBMS向けの標準規格言語であはあるものの、商品によって方言はある。製品仕様を必ず調べよう

## サブ言語
- データ定義言語（DDL）
  - CREATE, ALTER, DROP, ...
- データ操作言語（DML）
  - SELECT,....
<br>

__使うだけ__ に特化するのであれば、まずは後者だけ覚えよう

## 最初はこの12個だけでも調べて

1. SELCT : どんなカラム が必要？
2. FROM : どこのテーブル（ビュー）を使うの？
3. WHERE : 条件しぼる？（フィルターする？）
4. GROUP BY : グループ集計しよう
5. ORDER BY : 並び替える？
6. HAVING : GROUP BY後のWHEREと同じ機能
7. JOIN ON : テーブル同士を特定のキーで結合したい
8. WITH句 : クエリ分を見やすくするように、サブクエリは分割しよう
9. LIMIT : 一部だけ表示をしぼりたい（非推奨）
10. DISTINCT : ユニークな値のみ抽出
11. 各種集計指示 : COUNT(), AVG(), SUM()など
12. CASE文 : Excelでいうところのif文

- その他にも色々ありますが（窓関数とか）、適宜公式ドキュメント をググりましょう
- 本日は上記のうち、特に大切なものをご紹介

## BigQueryにアクセスしよう
- 画面で説明。BigQueryコンソロール画面に移動してください [リンク](https://console.cloud.google.com/bigquery?hl=ja)
- 注意：スキャン量に応じて課金されます
- LIMITを入れたからといって、課金は節約されません！（超大事）

## サンプルデータ
- [BQ FUN](https://bqfun.jp)のna0さん(twitter:na0fu3y)が下記の通りBQに法人番号データをアップロードしてくれています
- [加工済みオープンデータ](https://bqfun.jp/docs/jpdata/)
- 本日はここから、国税庁の[法人番号公表サイト]（https://www.houjin-bangou.nta.go.jp/ ）で練習する
- データ定義は[ここ](https://www.houjin-bangou.nta.go.jp/download/)を参照

## SELECTとFROMを理解しょう
- SELECTのあとに必要なカラム、FROMはデータのソースを定義するだけ（簡単！！）
```sql
SELECT
    col1  -- みたいカラムの名前
    col2,
    col3,
FROM
    `参照先のテーブル`
```
- 例えば
```sql
SELECT
  corporate_number,
  change_date,
  prefecture_name,
  city_name
FROM
  corporate_number_preprocessed_by_bq_fun.change_history
```

- 全てのカラム を持ってきたければ*で引っ張れる。
- ただし、SQLの可読性を損なう + 無駄なカラムのために課金消費するのは嫌。__非推奨！！__
```sql
SELECT
   *
FROM
  corporate_number_preprocessed_by_bq_fun.change_history
```
- ついでに、LIMIT 10を最後にいれると、１０レコードのい表示になる

## WHERE と ORDER BY
- 東京都狛江市だけのデータが欲しい
- 日付で並び替えたい

```sql
SELECT
  corporate_number,
  change_date,
  prefecture_name,
  city_name
FROM
  corporate_number_preprocessed_by_bq_fun.change_history
WHERE
  prefecture_name = "東京都"
  AND city_name = "狛江市"
ORDER BY
  change_date DE DESC
```

## GROUPごとに集計してみよう
- 年度ごとに法人登録された株式会社の数を集計してみる

```sql
SELECT
  EXTRACT(YEAR FROM change_date) AS start_year,  -- dateのYYYYだけ取り出す
  COUNT(DISTINCT corporate_number) AS new_cnt,
FROM
  corporate_number_preprocessed_by_bq_fun.change_history
WHERE
  process = "新規"
  AND kind = "株式会社" 
GROUP BY start_year
ORDER BY start_year
```
- （2015年に新規が多いのは本システムの都合上のもの）

- ついでに、`HAVING`について
- `GROUP BY`した結果に対して、さらにフィルターしたいときに使う

```sql
SELECT
　corporate_number,
  COUNT(DISTINCT name) AS name_cnt
FROM
  corporate_number_preprocessed_by_bq_fun.change_history
WHERE
  process = "新規"
  AND kind = "株式会社" 
GROUP BY corporate_number
HAVING 
  name_cnt > 1
```

## WITH句
- サブクエリ（FROMの中に、さらにSELECT文を書くようなもの）は第三者にとって非常に読みにくい。
- WITH句を積極的に活用しましょう
- (以下はあまり意味のないwith句ですが、練習として)

```sql
WITH start_up_corporates AS ( 
  SELECT
    DISTINCT corporate_number,
    change_date AS start_date,
    prefecture_name,
    city_name
  FROM
    corporate_number_preprocessed_by_bq_fun.change_history
  WHERE
    process = "新規"
    AND kind = "株式会社" 
    AND EXTRACT(YEAR FROM change_date) > 2016
  )

  SELECT
   *
  FROM 
    start_up_corporates
  WHERE
    city_name = "狛江市"
```

## JOIN
- table と tableをキーを基に、マージしよう
- start_up_corporates : 2016年以降に新規登録された株式会社
- close_corporates : 登録が消えた株式会社（見做しの廃業フラグとする）
- id_name : 法人番号及び最新の法人名
- 両テーブルをマージする
- `DECLARE`は定数を指定している（以下では本レコードの最大更新日）
- JOINのキー指定は`ON`でも良いが、テーブル間のキーの名前が同じであるため`USING`を使用した

```sql
DECLARE　_today DATE DEFAULT ( SELECT　MAX(change_date)　FROM　corporate_number_preprocessed_by_bq_fun.change_history );

WITH start_up_corporates AS (
  /*　この観察期間内における株式会社の設立日を取得 (baseテーブル)　*/
  SELECT
    DISTINCT corporate_number,
    change_date AS start_date,
    EXTRACT(YEAR FROM change_date) AS start_year,
    prefecture_name,
    city_name
  FROM
    corporate_number_preprocessed_by_bq_fun.change_history
  WHERE
    process = "新規"
    AND kind = "株式会社" 
    AND EXTRACT(YEAR FROM change_date) > 2016 -- 2015年のデータはバルクでデータアップロードされているものが混ざっている
),
close_corporates AS ( 
  /*　この観察期間内における株式会社の見做し消滅レコードを取得　*/
  SELECT
    DISTINCT corporate_number,
    1 AS event,
    process,
    close_date,
    close_cause,
  FROM
    corporate_number_preprocessed_by_bq_fun.change_history
  WHERE
    process IN ("登記記録の閉鎖等", "商号の登記の抹消")
    AND kind = "株式会社" ),
id_name AS (
  /*　法人番号に最新の名前を付与　*/
  SELECT
    DISTINCT corporate_number,
    LAST_VALUE(name) OVER(
      PARTITION BY corporate_number 
      ORDER BY change_date ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING 
    ) AS corporate_name,
  FROM
    corporate_number_preprocessed_by_bq_fun.change_history
)
/*　出力用のselect文　*/
SELECT
  corporate_number,
  i.corporate_name,
  IFNULL(c.event, 0) AS event,
  s.start_year,
  CASE
    WHEN c.close_date IS NULL THEN DATE_DIFF(_today, s.start_date, MONTH)
    ELSE　DATE_DIFF(c.close_date, s.start_date, MONTH)
  END　AS survival_month,
  s.start_date,
  s.prefecture_name,
  s.city_name,
  c.process,
  c.close_date,
  c.close_cause,
FROM
  start_up_corporates s
LEFT JOIN
  id_name i
USING
  (corporate_number)
LEFT JOIN
  close_corporates c
USING
  (corporate_number)
```

## LIMITつければ安心ではない
- 課金はあくあでスキャン量に依存する。LIMITはスキャンが終わってから実行される
- 本当のBIGDATAはパーティション（クラスター、index）がついてるので、かならずwhereでパーティションを指定してクエリを投げる（課金爆死する）

## view
- 適当なデータセットの下に、viewを作ってみよう
- viewはデータを物理的に複製しているのではなく、SQLクエリを保存しているようなもの
- viewを見るたびに、クエリ実行される
- tableの作成かviewの構築かは、スキャン量と利用頻度等を勘案しながら考えよう

## スケジュールクエリでテーブルを自動定期更新(便利機能)
- クエリを定期的に実行して、テーブルを上書き or 追加することお簡単

## 実務的には、
### チーム内で必ずルールを決めよう
- コーディング規約
- 特に、命名規約の徹底はしよう（スネークケース）
  - prefixで`raw_`(生データ)、`dwh_`(データウェアハウス)、`dtm_`(データマート)を識別する命名規則は本当におすすめ！是非やろう
    - raw : データソースのコピー（例外は個人情報などはハッシュ化・匿名化しておく）
    - dtm : アプリケーションと一対一対応したもの。例えば、tableauで分析する直前のデータなど。BIツールで複雑な前処理は事故の元なのでNG!!
    - dwh : raw -> dtmの構築の過程において、「あれ？この中間データ他でも必要なんじゃない？」と思ったときに外だしする中間データ。組織がどんどんBQを使い込むと良い感じのdwhが増えてくる
- table viewはsandbox以外、必ずドキュエントを書こう（どこに？フォーマットは？検索できることが大事）
- パーティション（クラスター）の設定・利用ルール
- 当然機微情報の管理なども！

## ASSERTION
- BQではAssert文も実装されている　[公式](https://cloud.google.com/bigquery/docs/reference/standard-sql/debugging-statements?hl=ja)
- 積極的に使おう

## 慣れてきたら効率的なクエリを書く
- [公式ドキュメント](https://cloud.google.com/bigquery/docs/best-practices-costs)を読んでみよう
  - SELECT *を避ける
    - 無駄なカラムのためにスキャン料金が発生するのはバカらしい
    - *は可読性を損ねる
  - パーティション(クラスター)を積極的に使う
    - 本当にデータが大きくなってきたとき、パーティション設定しないクエリは料金的に死ぬ
    - テーブルを作るときにパーティションがあるクエリしか受け付けないようにする防御策はある
  - テーブルを非正規化する
    - 意外かもしれませんが、BQの場合、非正規化しておいた方が効率的なクエリが実行できることが多々ある
    - 非正規化されたデータはJOINが必要ないので(列志向DBの特徴かと)
  - JOINする前にデータの量を減らす
  - クエリのキャッシュを有効利用する　などなど

## おまけ
### UDF(ユーザー定義関数)
- BigQueryはUDFという関数を作ることができる
- さらに、以下のようにGCSにjavascriptのライブラリーを格納することによって、javascriptのライブラリーを使用することもできる
- ご参考１：ymym3412さん [SQLで始める自然言語処理](https://ymym3412.hatenablog.com/entry/2020/12/24/001923)
- ご参考２：浅見 [BigQuery で統計処理を完結させる](https://lab.mo-t.com/blog/bigquery-udf)



