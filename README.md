air
===
"air"は東京都政府が提供するリアルタイムの大気質データを可視化するプロジェクトです。このプロジェクトの主要なコンポーネントは以下の通りです：
   * [www.kankyo.metro.tokyo.jp](http://www.kankyo.metro.tokyo.jp)から大気データを抽出するスクレーパー
   * データを保存するpostgresデータベース
   * このデータとその他の静的ファイルをクライアントに提供するexpress.jsサーバー
   * データを補間し、アニメーション化された風向マップをレンダリングするクライアントアプリ

"air"のインスタンスは http://air.nullschool.net で利用可能です。現在[Amazon Web Services](http://aws.amazon.com)と[CloudFlare](https://www.cloudflare.com)ででホスティングされています。

"air"は私がjavascript、node.js、when.js、postgres、D3、ブラウザ・プログラミングを学ぶために使用した個人プロジェクトです。設計上の決定の一部は、単純に新しいことを試すために行われました（例：postgres）。間違いなく、他の決定は経験不足から行われました。フィードバックをお待ちしています！


ビルドと起動
----------------------

プロジェクトをクローンして、npmからライブラリをインストール：

```
npm install
```

注意：[pg](https://github.com/brianc/node-postgres)をビルドするには[libpq](http://www.postgresql.org/docs/9.3/static/libpq.html)が必要です。libpqライブラリはMac OS Xではpostgresによって自動的にインストールされますが、AWSでは別途インストールが必要でした。

postgresをインストールしてデータベースを作成します。例えば：

```
CREATE DATABASE air
  WITH OWNER = postgres
       ENCODING = 'UTF8'
       TABLESPACE = pg_default
       LC_COLLATE = 'en_US.UTF-8'
       LC_CTYPE = 'en_US.UTF-8'
       CONNECTION LIMIT = -1;
```

サーバーを起動：

```
node server.js <port> <postgres-connection-string> <air-data-url>
```

例：

```
node server.js 8080 postgres://postgres:12345@localhost:5432/air <air-data-url>
```

最後に、ブラウザでサーバーにアクセスします：

```
http://localhost:8080
```

実装ノート
--------------------

このプロジェクトの構築には、いくつかの興味深い問題への解決策が必要でした。以下にいくつかを示します：

   * リアルタイム大気データはShift_JISでエンコードされたHTMLとして提供されています。Node.jsはネイティブでShift_JISをサポートしていないため、[iconv](https://github.com/bnoordhuis/node-iconv)ライブラリを使用してUTF-8への変換を行います。
   * 東京の地理データは国土交通省から直接入手し、80MBのXMLファイルでした。このデータは300KBの[TopoJSON](https://github.com/mbostock/topojson)ファイルに変換され、ブラウザがダウンロードして[D3](http://d3js.org/)でSVGとしてレンダリングできる十分に小さなサイズになりました。
   * 約50の観測所が毎時の風向と汚染物質データを提供しています。[逆距離重み付け](http://en.wikipedia.org/wiki/Inverse_distance_weighting)補間を使用して、東京全体をカバーする風ベクトル・フィールドを構築します。逆距離重み付けは、奇妙なアーティファクトを生成し時代遅れとされていますが、非常にシンプルでベクトル補間を実行するように拡張するのが簡単でした。
   * ブラウザは各点(x, y)をn個の最も近い観測所を使用して補間します。n個の最近傍を決定するために、クライアントは[k-d tree](http://en.wikipedia.org/wiki/K-d_tree)を構築し、これによりパフォーマンスが大幅に向上します。
   * 汚染物質の可視化オーバーレイは[Thin Plate Spline](http://en.wikipedia.org/wiki/Thin_plate_spline)補間を使用します。Thin Plate Splineは大気汚染物質などの自然現象に使用するには確実に間違った方法ですが、IDWよりも滑らかな表面を生成します。
   * 東京のSVGマップはHTML5 Canvasでオーバーレイされ、そこでアニメーションが描画されます。アニメーションレンダラーはSVGエンジンによって東京の境界がどこにレンダリングされるかを知る必要がありますが、このピクセル単位の情報をSVG要素から直接取得するのは困難です。この問題を回避するために、東京のポリゴンを分離されたCanvas要素に再レンダリングし、Canvasのピクセルがマスクとして動作して、マップ内にある点と外にある点を区別します。フィールドマスクは赤色チャンネルを占有し、ディスプレイマスクは緑色チャンネルにエンコードされます。
   * 楽しい実験だったので、ブラウザで[when.js](https://github.com/cujojs/when)を使用しました。

インスピレーション
-----------

[hint.fmの素晴らしい風向マップ](http://hint.fm/wind/)がこのプロジェクトの主なインスピレーションを提供しました。そして非常に優れたD3チュートリアル[Let's Make a Map](http://bost.ocks.org/mike/map/)が、いかに簡単に始められるかを示してくれました。
