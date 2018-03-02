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
// 20171010 START
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
// 20171010 END
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


## 検索結果画面にproduct_codeとかカテゴリを表示するカスタマイズ

<img src="images/検索結果画面のカスタマイズ01.jpg" alt="">

デフォルトの検索結果画面には品番(product_code)が表示されないので、品番での検索がメインの我々にとって非常に使いにくいものになっている。

以下、検索結果画面にproduct_codeを表示させるカスタマイズ。

### カスタマイズ内容

対象のソース

    /design/themes/basic/templates/views/products/components/one_product.tpl

追加コード

```smarty
{assign var="product_url" value="{"products.view?product_id=`$product.product_id`"|fn_url}"}
{assign var="product_url_arr" value="/"|explode:$product_url}
{assign var="product_url_count" value=$product_url_arr|@count-2}
{assign var="product_url_category" value=$product_url_arr[3]}
<!-- 表示 -->
<small>{$product_url_category|upper} / {$product_url_arr[$product_url_count]|upper}</small>
```

URLを`/`でパースして分解、カウント末尾の値を取得して表示しています。
おまけでカテゴリも表示しています。

### なぜこのような回りくどい取り方をするのか

`{debug}`で試すとわかりますが、デフォルトでは`product_code`はおろか、頼みの`seo_name`すら含まれていません。

`product_code`あるいは`seo_name`を取得するためにはコアへの手入れが必須であるため、それを避けた結果がURLのパースです。

URLという相対的なデータを元に重要な機能を実装するのはハイリスクなので、一応こうしてドキュメントだけ残しておきます。


## CS-CART4でのCMS利用（任意HTMLの記述）について

HTMLを記述する際の注意事項です。

### WYSIWYGエディタについて

CS-CARTでは以下の3種類からWYSIWYGエディタを選択できます。

1. CKEditor
2. Reductor
3. TinyMCE

CMS機能で任意のHTMLを記述する際、WYSIWYGエディタによっては記述したとおりにHTMLが保存されず、例えば勝手に`<br>`や`<p>`が差し込まれたり、値が空のタグが勝手に削除されたり、ヒドイときにはid属性が消えたりします。

結論から言うと、上記3つの中で実用的なのはTinyMCEのみです。残りの2つは要素の削除と属性の削除が発生します。

TinyMCEの場合でも以下の点に気をつける必要があります。

値が空のタグは削除されることがあります。
: 例えば`<i class="icon-home"></i>`など。

タブ文字や要素の前後のスペースが全て削除されます。
: 簡単に言うと整形が無くなります。

また、以下のメリットがあります。

scriptタグが使える
: 記述したスクリプトタグはCDATA化され自動でbody下部に移動します。（超賢い）


## 注文処理画面に本来の品番を出力する（コンビネーション運用時）

コンビネーションコードにJANコードを入力する運用において、あらゆる箇所でプロダクト本来の品番が表示されず、コンビネーションコードが出力されてしまう問題を解決するために行った改修。

### 経緯

注文処理画面や、注文メール、納品書などのPDFで型番（品番）が表示されない。まずフロント側の大部分では`seo_name`の表示等によって回避していた問題だが、注文データからの紐付きにおいて品番がどうしても取れず、受発注などの運用に支障をきたすことが判明した。  
（`var_dump`等であらゆる変数を出力したが、product_codeにはJANが設定されてしまっていた）

クエリを追うと、そもそも品番を取得していないことが判明した。

仕方なく品番を取得できるようにコアを改修することとなった。

### コアの改修

#### fn.cart.phpの修正

function fn_get_order_info内

```php
$order['products'] = db_get_hash_array(
    "SELECT ?:order_details.*, ?:product_descriptions.product, ?:products.status as product_status FROM ?:order_details "
    . "LEFT JOIN ?:product_descriptions ON ?:order_details.product_id = ?:product_descriptions.product_id AND ?:product_descriptions.lang_code = ?s "
    . "LEFT JOIN ?:products ON ?:order_details.product_id = ?:products.product_id "
    . "WHERE ?:order_details.order_id = ?i ORDER BY ?:product_descriptions.product",
    'item_id', $lang_code, $order_id
);
```

↓↓↓

```php
$order['products'] = db_get_hash_array(
    //"SELECT ?:order_details.*, ?:product_descriptions.product, ?:products.status as product_status FROM ?:order_details "
    "SELECT ?:order_details.*, ?:product_descriptions.product, ?:products.status as product_status, ?:products.product_code as hinban FROM ?:order_details "
    . "LEFT JOIN ?:product_descriptions ON ?:order_details.product_id = ?:product_descriptions.product_id AND ?:product_descriptions.lang_code = ?s "
    . "LEFT JOIN ?:products ON ?:order_details.product_id = ?:products.product_id "
    . "WHERE ?:order_details.order_id = ?i ORDER BY ?:product_descriptions.product",
    'item_id', $lang_code, $order_id
);
```

`products.product_code`をselectに含め、別名`hinban`とした。

#### 問題発生

`hinban`を取得できるようになったものの、あらゆる箇所（注文管理画面や注文確認書印刷など）で`hinban`の出力が求められるため、コアのあらゆる箇所に手入れが必要になった。

これでは保守が現実的ではないため、他の手段を模索することにした。

すぐに思いつくのは、既存のフィールドに`hinban`の文字列をリテラル連結してしまうことだ。

例えば商品名(product)フィールドに含めてしまうなど。

### 商品名の頭にhinbanをリテラル連結する

fn.cart.php

1868行目付近

```php
//$order['products'][$k]['product'] = fn_get_product_name($v['product_id'], $lang_code);
$order['products'][$k]['product'] = $order['products'][$k]['hinban'] . ' - ' . $v['extra']['product'];
```

OK。事なきを得た。


## CS-CARTのCSVエクスポート項目を増やす

### 目的

CS-CARTのコンビネーションCSVをエクスポートする時、`product_code`もレイアウトに含めたい。

### 経緯

在庫数やJANコードを更新するにはコンビネーションcsvをエクスポートし、値を書き換えたあとインポートするが、レコードのマッチングには品番(`product_code`)が必要だが上記CSVのレイアウトには含まれていない。（含まれているのは`product_id`）

`product_code`を得るためには、コンビネーションcsvと商品情報csvをリレーションしなければならなかった。これが運用上ネックになっていた。

### 方法

    /schemas/exim/product_combinations.php

上記ファイルでエクスポートのレイアウト情報が管理されていた。

    'Product code' => array(
        'required' => false,
        'table' => 'products',
        'db_field' => 'product_code'
    ),

上記コードを差し込むだけで希望の動作を得ることができた。


## カテゴリー画面で品番からIDを得るスクリプト

ショップの運用上、管理コード`product_code`と、CS-CARTでインクリメントされている`product_id`は異なるものになります。

`product_code`の一覧から`product_id`を得るにはCSVをダウンロードするしか無いのですが、面倒なのでscriptで解決します。

~~~js
// 品番の配列
var items = [
'product_code1'
,'product_code2'
,'product_code3'
,'product_code4'
,'product_code1'
,'product_code2'
,'product_code3'
,'product_code4'
,'product_code5'
,'product_code1'
];
var result = "";
$.each(items, function(i,v){
  var targetInput = $('.product-code input.input-hidden[value=' + items[i] + ']');
  var targetAnchor = $(targetInput).parent().parent().find('.row-status');
  result += $(targetAnchor).attr('href').split('product_id=')[1] + '\n';
});
// 出力結果をcombinations.csvのProduct ID列にコピペ
console.log(result);
~~~

1. items変数に、idを得たいcodeを羅列して下さい。（重複OK）
2. 上記scriptを、管理画面の商品一覧画面、コンソールで実行して下さい。（idは画面上に存在する商品しか取れません）
3. 出力結果をコピーして下さい。

### カテゴリー画面でポジション(並び順)を入力するスクリプト

商品の並び順はカテゴリー毎に制御する必要があります。  
CSVで管理できないのでインターフェースは管理画面か直接DBを叩くしかありません。

管理画面のカテゴリー内商品一覧画面で、以下のscriptを発行すると手入力を省けます。

~~~js
// "品番":"並び順"のJSON
var items = {
"product_code1":"1",
"product_code2":"2",
"product_code3":"3",
"product_code4":"4",
"product_code5":"5"
};
$('.product-code .product-code-label').next('input').each(function(i,v){
  var c = $(v).val();
  if(c in items == true){
    $(this).parent().parent().parent('tr').find('td:nth-child(2)').find('input.input-micro').val(items[c]);
  } else {
    console.log("失敗 : " + c);
  }
});
~~~


## CS-CARTの管理画面の商品一覧で、指定した品番だけチェックを入れるやつ

以下のscriptを発行すると、任意の品番にチェックを入れられます。

```js
var targetCodeArr = [
'product_code1'
,'product_code2'
,'product_code3'
,'product_code4'
,'product_code5'
];
var checkTargetCodes = function(){
  console.log('比較開始');
  var itemCount = 0;
  for (var i=0;i<targetCodeArr.length;i++){
    $('.product-code input').each(function(index,target){
      var targetCode = $(target).val();
      if(targetCode == targetCodeArr[i]){
        $(target).parent().parent().parent().find('.left .checkbox').prop("checked",true);
        console.log(targetCode);
        itemCount++;
      }
    });
  }
  console.log('処理した品番数：' + itemCount + ' / ' + targetCodeArr.length);
}();
```
