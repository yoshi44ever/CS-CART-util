# CS-CART管理保守で使うユーティリティscript集とtips集

CS-CARTデフォルトの管理機能では管理に手作業が頻発するので、補助sciptを書いています。

## カタログモードで販売しない商品に対し発行するSQL

カタログモードで使用する`buy_now_url`はインポートCSVから指定できない。  
管理画面からポチポチ入力するのは現実的ではないため、SQLで一括指定する。

```sql
update cscart_products set
buy_now_url = '#',
tracking = 'D'
where product_code in (
'product_code1'
,'product_code2'
,'product_code3'
)
```

`buy_now_url`の右辺に任意の値を設定する。
`product_code$`の部分は対象の品番を羅列する。  
`tracking = 'D'`は在庫管理しない設定。

### 逆に販売対象にしたい場合

~~~sql
update cscart_products set
buy_now_url = '',
tracking = 'O'
where product_code in (
'product_code1'
,'product_code2'
,'product_code3'
)
~~~

## ブログを同日日付で複数投稿しても、最新順にならない。

CS-CARTデフォルトのブログ機能は稚拙なものであり、管理運用上トラブルが頻発します。  
特に同日日付の並び順が制御できない問題は深刻です。

### 原因

強制的に00:00:00で登録されてしまうため。

> app/functions/fn.common.php
> function fn_parse_date

上記処理で強制的に00:00:00あるいは23:59:59に書き換えられてしまう。

### 悪あがき

まずビュー側からパラメータを取得した時点で00:00:00になっているので、

> app/addons/blog/func.php
> function fn_blog_update_page_post

timestampパラメータUNIXタイムに時分秒を追加してみる。

```php
<?php
////////////////////////////////////////////////////////////
// 20171010 Niiseki START
////////////////////////////////////////////////////////////
// 新規作成時のみ
if($create){
    if(ctype_digit($page_data['timestamp'])) {
        // 現在h:m:sのUnixTimeを取得
        $hms = TIME - mktime(0,0,0,date("n"),date("j"),date("Y"));
        // パラメータのtimestampに加算
        $page_data['timestamp'] = intval($page_data['timestamp']) + $hms;
    }
}
////////////////////////////////////////////////////////////
// 20171010 Niiseki END
////////////////////////////////////////////////////////////
?>
```

結局上記の`fn_parse_date`で丸められるので意味はありませんでした。

解決策としては、良いアドオンを見つけるくらいしか無いと思います。

## CSVで画像をインポートする際のtips

CSVインポートの際、画像を特定のディレクトリに配置して、特定のURLを記述すると一括インポートが可能となる。

### 画像をアップロードするディレクトリ

```
/var/files/1/exim/backup/images
```

### CSVに記述するURL例

```
https://yoursite.com/var/files/1/exim/backup/images/product_code.png
```

### 注意点

CS-CARTではcombination（商品オプション）の掛け合わせ毎に画像を持ちますが、それらは全て別の物理ファイルでなければならない仕様があります。  
combinationを一括インポートする際、同じ画像を指定しても、`_..-....`が末尾に付加されたコピー画像が量産されます。  
全くもって不便で、オプションが多い商品（例えばアパレルなど）を運用する用途にはCS-CARTは不向きです。


## 特定商品が割引になるクーポンの発行時、対象商品をDBから一括登録する方法

**「一部の商品をクーポンで割引になるようにして欲しい」**

キャンペーン機能で可能ですが「一部の商品」が数百となると手動入力は実質不可能です。

なのでデータベースに直接一括登録する方法を紹介します。

### 手順

マーケティング＞キャンペーンを開き、新規キャンペーンを追加してください。

「条件」にクーポンコードを設定してください。

「特典」に「特定商品を割引」を設定し、適当に１つ商品を追加してください。

割引率を設定してください。

phpMyAdminで、以下のテーブルを開いてください。

`cscart_promotions`

インラインで以下のSQLを発行してください。

```sql
SELECT *
FROM `cscart_promotions`
WHERE promotion_id = [プロモーションID]
```

対象ページで有効になっているブロックが行で現れます。

`conditions`フィールドに以下の内容を入力してください。

```
a:1:{i:2;a:4:{s:5:"bonus";s:20:"discount_on_products";s:5:"value";s:605:"157,161,164,167,174,184,188,189,191,193,194,196,203,205,206,211,212,219,220,221,235,236,252,253,256,258,259,260,261,262,263,264,265,266,267,268,269,270,271,272,273,274,275,276,277,278,279,280,281,282,283,285,292,293,294,295,314,320,321,323,327,328,329,330,331,332,333,334,335,336,337,339,340,341,344,346,347,1491,559,583,585,588,591,592,593,596,597,598,606,607,613,614,615,617,618,619,620,621,622,623,624,626,627,629,635,657,661,662,663,664,665,666,667,668,669,670,671,672,673,674,675,676,680,686,691,704,708,737,738,743,744,745,747,748,750,763,770,777,779,1393,1396,368,369,382,385,367,150,146,148,149,57";s:14:"discount_bonus";s:13:"to_percentage";s:14:"discount_value";d:50;}}
```

数値の羅列は商品IDです。  
`"value";`の次の`s:605`は対象商品ID数に4を掛けたものか、それに-1か+1したものです。  
商品数が100なら400 or 399 or 401だと思います。変えながらページをリロードして、画面に反映されればその数です。数式的なロジックは調べていません。

対象商品品番のIDを得るには、`cscart_products`テーブルで以下のようなSQLを発行してください。

```sql
SELECT * FROM `cscart_products` where product_code in (
'product_code1'
,'product_code2'
,'product_code3'
,'product_code4'
,'product_code5'
,'product_code6'
,'product_code7'
,'product_code8'
,'product_code9'
)
```

抽出結果を全行表示し、エクスポートするなりテキストコピーするなりして対象品番を取得してください。


## フックの使い方

CS-CARTはパッケージソフトウェアであるため、細かい需要に仕様が対応していないことが多いです。

そういった場合は当然カスタマイズするわけですが、コアに近い部分を改修してしまうと、セキュリティアップデートなどのときに上書きされてしうことがあります。

そういった自体を避けるために、コアのあちこちに「フック」というものが仕掛けられており、そのフックを利用することでコアの機能に処理を差し挟む、あるいは処理を乗っ取ることが出来ます。

### フックとはどういうものか

例えば、

    /app/functions/fn.catalog.php

上記ファイルの、

~~~php
<?php
function fn_get_feature_name($feature_id, $lang_code = CART_LANGUAGE, $as_array = false)
{
    fn_set_hook('get_feature_name_pre', $feature_id, $lang_code, $as_array);

    $result = false;
    if (!empty($feature_id)) {
        if (!is_array($feature_id) && strpos($feature_id, ',') !== false) {
            $feature = explode(',', $feature_id);
        }

        $field_list = 'fd.feature_id as feature_id, fd.description as feature';
        $join = '';
        if (is_array($feature_id) || $as_array == true) {
            $condition = db_quote(' AND fd.feature_id IN (?n) AND fd.lang_code = ?s', $feature_id, $lang_code);
        } else {
            $condition = db_quote(' AND fd.feature_id = ?i AND fd.lang_code = ?s', $feature_id, $lang_code);
        }

        fn_set_hook('get_feature_name', $feature_id, $lang_code, $as_array, $field_list, $join, $condition);

        $result = db_get_hash_single_array("SELECT $field_list FROM ?:product_features_descriptions fd $join WHERE 1 $condition", array('feature_id', 'feature'));
        if (!(is_array($feature_id) || $as_array == true)) {
            if (isset($result[$feature_id])) {
                $result = $result[$feature_id];
            } else {
                $result = null;
            }
        }
    }

    fn_set_hook('get_feature_name_post', $feature_id, $lang_code, $as_array, $result);

    return $result;
}
~~~
上記の関数はidを渡すと1フィールドの結果を返すシンプルな関数ですが、この短い処理の間に3つのフックが仕掛けられています。

    fn_set_hook('get_feature_name_pre', $feature_id, $lang_code, $as_array);
    fn_set_hook('get_feature_name', $feature_id, $lang_code, $as_array, $field_list, $join, $condition);
    fn_set_hook('get_feature_name_post', $feature_id, $lang_code, $as_array, $result);

タイミングとしてはそれぞれ

1. 処理の開始前
2. SQLを発行して結果セットを取得する直前
3. 値をリターンする直前

となっています。これは非常に親切にフックが仕掛けられています。
ここまで細かくフックが仕掛けられていれば、間違いなく任意の処理に書き換えることができます。

次にフックの使い方です。

### フックの利用宣言

アドオンのinit.phpに以下のように記述します。

~~~php
fn_register_hooks(
    'get_product_name'
);
~~~

上記は`get_product_name`というフックを利用するという宣言です。

## 関数の宣言

アドオンのfunc.phpに以下の規則で関数を宣言します。

    funcition アドオン名_フック名称(引数) {}

上記に規則で宣言するだけでフックが自動的に機能します。

当然、参照渡しで引数を渡せば、元の値を書き換えられます。

    funcition アドオン名_フック名称(&$name) {
      $name = '上書き！';
      return false;
    }

### フックの挙動

例えば下記の関数があったとします。

~~~php
function sampleHook() {
  $num = 0;
  fn_set_hook('sample_hook_1', $num);
  $num = 1;
  fn_set_hook('sample_hook_2', $num);
  $num = 2;
  fn_set_hook('sample_hook_3', $num);
  return $num;
}
~~~

上記に対し、以下のようにフックを利用したとします。
~~~php
function fn_myaddon_sample_hook_2(&$num) {
  $num = 3;
  return false;
}
~~~

すると、最終的に`sampleHook()`がreturnする値は`3`でしょうか？それとも`2`でしょうか？

答えは`3`になります。`$num = 2;`という行が無視されています。

フックはそのフックの開始時点から、次のフック、あるいは関数の終端までを乗っ取る挙動を示します。


