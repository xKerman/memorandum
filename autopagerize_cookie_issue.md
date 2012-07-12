# AutoPagerize における Cookie の取り扱いについて #


およそ1年前のことになるけれど、
AutoPagerize が次ページを継ぎ足す際に送信する Cookie の扱いにセキュリティ上の問題があることを発見して、
swdyh さんに連絡して修正された、ということがあった。
今回はそのことについて書く。

念のために書いておくと、
Chrome や Safari 向けの AutoPagerize には特に関係ない話
（iframe を使ってリクエストしているので）。
また、 Greasemonkey スクリプトの方も Firefox 拡張の方も、
ここに書く問題は既に修正されている。
なので利用者が特に心配することはない、と思う。
もちろん、古いのをうっかりアップデートせずに使っている人が仮にいるとすれば、
早めに更新した方がいい。


## 概要 ##
以下の2つの問題があった。

1. 本来 `https://` の URL へのリクエスト時にのみ送信されるはずの secure 属性のついた Cookie を、 `http://` の URL に対するリクエスト時にも送ってしまう問題
2. 送信先のドメインとはと別のドメインの Cookie を送信してしまう問題

AutoPagerize の Greasemonkey スクリプトには 1. の問題が、
Firefox 拡張には 1. および 2. の問題があった。
あとは uAutoPagerize にも 1. の問題があったけれど、
こちらも同時期に報告して修正済み。

アップデートの告知などは以下のとおり。

* AutoPagerize for Greasemonkey: <http://autopagerize.tumblr.com/post/7115480120/updates>
* AutoPagerize for Firefox: <http://autopagerize.tumblr.com/post/7115480120/updates>, <http://autopagerize.tumblr.com/post/7153030424/updates>
* uAutoPagerize: <http://d.hatena.ne.jp/Griever/20110704/1309772243>


### 1\. について ###
GM_xmlhttpRequest や Chrome 権限での XHR は、
「サードパーティのCookieも保存する」のチェックを外していると
リクエスト時に Cookie を送信しないという性質がある。
そのため、こういった種類の XHR を利用して認証が必要なページを読み取ろうとしても、
認証情報を含んだ Cookie が送信されないためにそのページを読み取ることができない。
AutoPagerize の場合だと認証を必要とするページの継ぎ足しができないことになる。

サードパーティ Cookie をオフにしてるとページの継ぎ足しに支障があるというのは不便なので、
Greasemonkey スクリプトや Firefox 拡張の AutoPagerize では、
XHR でのリクエスト時に `document.cookie` の情報をヘッダに追加するようにしている。
こういった形で Firefox 向けの AutoPagerize ではリクエスト時の Cookie を制御していた、というのが今回の話の前提知識。

さて、 Cookie には secure 属性がついている場合がある。
この secure 属性がついた Cookie の情報は `https://` の URL へのリクエスト時には送られるが、
`http://` の URL に対するリクエストの際には送信されない、という性質を持つ。
ここで例えば `https://example.com/` のページに対して適用される SITEINFO によって
次のページの URL として `http://example.com/` が選ばれた場合を考える。
このとき AutoPagerize は `document.cookie` で送信する Cookie の情報を得ていたのだけれど、
その情報の中には secure 属性がついた Cookie も含まれる。
これがそのまま `http://example.com/` へのリクエストで送られてしまったために、
`https://` でない URL のリクエストの際に secure 属性付きの Cookie が送信されてしまっていた。

これの何が困るかというと、
通常 secure 属性付きの Cookie は `https://` の暗号化された経路でのみ送られることから、
セッション ID などの途中経路で盗聴されては困る情報の送信に用いられるのだけれど、
AutoPagerize が secure 属性付き Cookie を `http://` の URL に対して送ってしまうと経路上で暗号化されていないために盗聴可能になってしまう。
なのでセッションハイジャックに悪用される可能性はあったと思う。

`document.cookie` で得られる `key1=value1; key2=value2` という key-value ペアのうち、
どのペアに secure 属性がついているのかを判定する方法はないので、
`document.cookie` の情報をリクエストヘッダに追加する際には、
送信元と送信先のドメインの一致を確認することに加えて
送信元が `https://` である場合は送信先も `https://` の場合に限って
ヘッダに追加することが必要になるだろう。
スキーム + ドメインが一致するかどうかを確認する、という方が楽かもしれない
（実際そういう風に修正された）。


### 2\. について ###
Cookie は基本的にドメインが同じでなければ送信されないので、
`document.cookie` の情報を XHR のヘッダに追加する際には
送信元と送信先が同一ドメインなのかを確認する必要がある。
このチェックが漏れていたことが原因で、
例えば `http://d.hatena.ne.jp/eviluser/` のページで AutoPagerize が次のページの URL として `http://logger.example.com/` を拾った場合に、
`d.hatena.ne.jp` の Cookie が `logger.example.com` に対して送信される、ということがありえた。


--------------------------------------------------------------------------------


## 余談： SITEINFO の優先度を上げる ##
上述した問題を悪用する場合、
ページ上に存在する特定の URL を AutoPagerize に次のページのものとして拾わせて
リクエストさせる必要がある。
そうすると、そのページ用に悪意を持って用意した SITEINFO が優先的にマッチするような状況を作る必要が出てくる。
ただ、あるページにマッチする SITEINFO は複数存在することが考えられるため、
優先順位を高く設定する必要があるだろう。
そのためにどうすればよいかといえば、
AutoPagerize は SITEINFO の `url` の値が長い方を優先的に用いるので、
`url` の正規表現を任意に長くできればよいことになる。

例えば上述の問題を利用して `https://example.com/` から `http://example.com/` に対して AutoPagerize によるリクエストを送信させて secure 属性付き cookie を盗聴する場合、

    {
        "url": "^https://example\\.com/.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?.?",
        "nextLink": "//a[starts-with(@href, \"http://example.com/\")]",
        "pageElement": "//nosuchelement"
    }

といった SITEINFO を登録することで、
仮に `https://example.com/` に対する SITEINFO が既に登録されていたとしても、
こちらの方が優先的に適用され、結果として盗聴が成功するだろう
（ついでに書くと `//nosuchelement` のようにページ上にに存在しない要素を指定することでリクエストは発生するものの継ぎ足しは行われない、
という状況を作り出せるので若干バレにくくなるかもしれない）。

まあ、 secure 属性付き cookie を盗聴する場合は途中経路で悪意のある SITEINFO を含んだ JSON ファイルと差し替えることができると思うので、こういう小手先のテクニックが使われるとは考えにくいけれど。


## 余談2： 現時点での Firefox 拡張版 AutoPagerize での cookie の扱い ##
`document.cookie` では httponly 属性のついた cookie の情報が取得できず、
そのために認証が必要な一部ページ (<http://www.tumblr.com/dashboard/> など) で次のページを継ぎ足せない場合があった。
そのため現在では <https://github.com/swdyh/autopagerize_for_firefox/pull/9/files> のように Chrome 権限のスクリプト経由で Cookie の情報を （httponly 属性がついたものも含めて） XHR のリクエストヘッダに追加するようになっている。

もちろんこれは拡張だからできることで、 Greasemonkey スクリプトではこういう芸当は無理。
