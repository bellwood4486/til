https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent

> フォーム POST からサーバーに渡されるユーザー入力値には encodeURIComponent を使用します。これは、特殊な HTML エンティティやエンコード・デコードを必要とする他の文字のデータ入力中に誤って生成される可能性がある "&" シンボルをエンコードします。

Content-Typeが`application/x-www-form-urlencoded`のリクエストのペイロードに含める場合、
`encodeURIComponent`でエンコードする。
`encodeURI`とは違うので注意。
