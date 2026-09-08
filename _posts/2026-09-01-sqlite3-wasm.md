---
layout: post
title: "WebでSQLite3を使う"
date: 2026-08-31 22:00:00 +0900
last_modified_at: 2026-08-31
description: SQLite3用のWebAssemblyを使ってみました。OPFS領域を使えばデータの永続化も可能です。
img: 2026/2026-08-31-1.png # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [database,sqlite]
categories: 
  - database
---

ブラウザでアプリケーションを作成していて、ローカルでSQLiteを動かしたいと思い、調べてみたら公式の[SQLite Wasm](https://github.com/sqlite/sqlite-wasm)を使うと実現できそうなので、使ってみることにしました。

以降で書かれている使い方を少し拡張した[サンプル的なプログラム](https://github.com/sugakenn/blog_box/tree/main/sqlite3-wasm)の方もよければ参考にしてください。


## ブラウザの特殊なファイル保存領域

基本的にはブラウザはセキュリティを担保する目的から、ローカルの端末資源にはアクセスできないようになっています。

そのため、SQLite Wasmが操作するファイルはそのブラウザ専用の領域に限られます。

ブラウザ専用の領域は`OPFS(Origin Private File System)`と呼ばれます。Originと名前がつけられている通り、オリジン単位でデータが保存されます。

この領域を使わない方法でもSQLite Wasmを動かすことはできますが、その際はデータは永続化できません。

ここに保存したデータベースはアプリの機能でエクスポートができます。また既存のSQLiteデータベースのインポートも可能です。

ちなみに、そのようなブラウザ固有の保存領域には、`localStorage`や`IndexedDB`などもあります。数個のデータを保持したいだけなら`localStorage`、SQLを発行する必要がなければ`IndexedDB`に保存した方が扱いは楽です。

sqlite-wasmの`Wasm`の部分ですが、これは(**W**eb**As**se**m**bly)の略です。C++などで書かれたバイナリをWebで高速で動かす仕組みです。

## 要件

[Web Worker](https://developer.mozilla.org/ja/docs/Web/API/Web_Workers_API)は、時間のかかる処理をバックグラウンドで行うJavaScriptのWeb APIです。

通常OPFSを利用するのにはWeb Workerは必須ではありませんが、SQLite Wasmがブラウザ内部のデータベースファイル管理をするのにWeb Workerでのみ利用可能な機能を使用しているためで、データを永続化したいなら`Web Woker API`を使う必要があります。

また、次のHTTPヘッダの出力が必要です。

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

SQLiteの公式ページ[Achtung: COOP and COEP HTTP Headers](https://sqlite.org/wasm/doc/92e1d3dab4/persistence.md#coop-coep)に詳しい事情が書いてありますが、`SharedArrayBuffer`を利用するためだという事です。

このヘッダを出力すると、CDNなどオリジンの違う場所からライブラリを取得することができなくなるので、注意が必要です。


## JavaScriptの違い

少し話がそれますが、今後コードを書いていく上では重要になるJavaScriptの違いについて書きます。

`ECMAScript`と`TypeScript`に区別されたり、`ESM(ES Modlues)`か`CommonJS`に区別されたりする時の違いの話です。

JavaScriptは、基本的に`ECMAScript`という規格によって言語の文法や動作が定められています。


`TypeScript`は、そのECMAScriptに型の概念を加える等の拡張が加えられた言語仕様です。

`ESM(.mjs)`か`CommonJS(.cjs)`かという際の違いは、JavaScriptのモジュール仕様の違いです。
具体的には`import`を使うのがESM、`require`を使うのがCommonJSとなります。そして、後者はECMAScriptの規格外の仕様になりますが、[Node.js](https://nodejs.org/)などで古くから使われてきた経緯があり現在も仕様が残っています。

そして、今回出てきた`OPFS`や`Web Worker`も狭義のJavaScript(≒ECMAScript)の範疇には含まれず、`Web API`として定義されているものです。またこのWeb APIにはDOM(document.xxx)も含まれます。Node.jsにおいて`document.`が使えないのは、Node.jsではWeb APIが提供されていないからです。
同様にブラウザ側で`fs.`が使えないももブラウザが`Node API`を提供していないからです。

```
         ECMAScript
             │
   ┌────┴─────┐
   │                    │
Browser               Node.js
   │                    │
Web API               Node API
   │                    │
document               process
fetch                  fs
IndexedDB              path
OPFS                    etc.
```

もうひとつ、HTML上に記述したり読み込んだりする従来のスクリプト(Classic Script)と、モジュールとして読み込まれる(ESM)の違いについて触れておきます。

| 項目                   | Classic Script | ES Module       |
| -------------------- | -------------- | --------------- |
| DOMアクセス              | ○              | ○               |
| `window`             | ○              | ○               |
| `document`           | ○              | ○               |
| `import` / `export`  | ×              | ○               |
| 自動的にstrict mode      | ×              | **○**           |
| トップレベル変数が`window`に入る | ○※1             | **×**           |
| HTML解析との実行順          | 通常即時           | `defer`相当       |
| `this`（トップレベル）       | `window`       | **`undefined`** |
| CORS※2                 | 通常scriptより緩い   | **CORS対象**      |

`※1`Classic Scriptは、`var`や`function`をトップレベルへ定義した際に、windowに変数が入りますが、`let`、`const`といった値は入りません。
ただ、トップレベルで定義した`let`、`const`は、グローバル変数/定数として認識できます。


一方Moduleのトップレベルで書いた変数は、意図してexportしない限り、他のモジュールからは見えません。
また、`export`されたものでもClassic Scriptからは読めないので、その際はESM側でエクスポートしたい値や関数をwindowsのプロパティにセットします。

`※2`Classic Scriptでは`<script src="...">`として別オリジンのソースを読み込めますが、moduleで読み込む場合は、取得先が返すHTTPヘッダの`Access-Control-Allow-Origin`が一致しなければ読み込まれません。

最後に、JavaScriptには`Angular`や`React`、`jQuery`、`Vue`といった、フレームワークがありますが、それらを使わないピュアなJavaScriptという意味で、`vanilla(バニラ) JavaScript`という言葉があります。


## Sqlite Wasmの設定

それでは実際にWeb設定していきます。SQLite公式ページ[WebAssembly & JavaScript](https://sqlite.org/download.html)から、`sqlite-wasm-xxxxx.zip`ファイルをダウンロードします。

展開したディレクトリの中の`jswasm`サブディレクトリには次のようなファイルが入っています。
|名前|説明|
|:-|:-|
|sqlite3.wasm|コアファイルです|
|sqlite3.mjs|モジュールとして使用する際に読み込むファイルです。<br>これがsqlite3.wasmを読み込みます。|
|sqlite3.js|モジュールとして使わわない場合は、こちらを使いますが今回は使いません。|
|sqlite3-opfs-async-proxy.js|OPFSを使う場合に自動で読み込まれます。|

他のファイルも状況によって使うことがあるので、まとめてサーバーの同一レベルに配置しておきます。
今回は`worker`を自作するので、先のテーブルの`sqlite3.js`ファイルを除いた3つのファイルだけを使うことになります。


## Workerの作成

WorkerはWebページに所属するメインのコードで生成されますが、そうして生成されたオブジェクトの内部からはDOMに直接触れることができません。

Workerに対して、`postMessage`でデータを渡し、処理の結果を`onmessage`イベントで受け取るという使い方をします。

Worker内部のコードでも、onmessageでデータを受け取り、postMessageでメイン側にレスポンスを返します。これらの処理は非同期的に行われます。

長大になると管理しづらくなるので、Workerのコードはメインページ毎や機能毎にモジュール化するといいと思います。

```javascript
// sqlite3-worker.js
// import時./はファイルのある場所を基準にする
import sqlite3InitModule from "./sqlite3.mjs";

//初期化されるまでブロック
const sqlite3 = await sqlite3InitModule();

let db = null;

self.onmessage = async event => {

    /**
     * @param id { int } メッセージのid これを使って、レスポンスを結びつける
     * @param owerPage { string } ページ毎にDBを管理するのでその識別子
     * @param type { string } 処理の区分
     * @param data { * } 受けとるデータ
     *
     */
    const {
        id,
        ownerPage,
        type,
        data
    } = event.data;

    try {
        if (!ownerPage) {
            //想定外
            throw new Error(
                `オーナーページが設定されていません(想定外)`
            );
        }
        
        const ownerPageWithoutExt = ownerPage.replace(/\.[^.]+$/, "");
        
        //データベースのイニシャライズは共通処理
        if (type === "initdb") {
            const dbName = `app.${ownerPageWithoutExt}.sqlite3`;

            db = new sqlite3.oo1.OpfsDb(
                `/${dbName}`,
                "c"
            );

            self.postMessage({
                id,
                success: true
            });

            return;
        } 

        if (!db) {
            throw new Error("DBが初期化されていません");
        }

        
        //ページ毎での各実装へ
        const moduleUrl = "./submodules/" + ownerPageWithoutExt+ ".mjs";

        if (!moduleUrl) {
            console.error(`ページ毎のDB処理定義がありません: ${ownerPageWithoutExt}`);
            throw new Error(
                `ページ毎のDB処理定義がありません: ${ownerPageWithoutExt}`
            );
        }
        
        //サブモジュールを読み込み
        const module = await import(moduleUrl);

        //サブモジュールはexecute関数を実装する前提
        if (typeof module.execute !== "function") {
            console.error(`execute() is not defined in ${moduleUrl}`);
            throw new Error(
                `execute() is not defined: ${ownerPageWithoutExt}`
            );
        }

        //サブモジュールの処理を実行、実行結果を返す、エラー時は例外を投げる
        const result = await module.execute(
            db,
            type,
            data
        );
        
        //処理が終わったらメインへ結果を返す
        self.postMessage({
            id : id,
            success: true,
            data: result
        });

    } catch (error) {
        //エラー処理
        self.postMessage({
            id : id,
            success: false,
            error: error.message
        });
    }
};

//SQLite3の読み込み終了とWORKER待機開始を通知する 
self.postMessage({
    id: "initworker",
    success: true,
    message: "sqlite-worker.js is ready"
});
```

上記のWorker用のスクリプトを作ったら、メイン側ではそのパスをWorkerのコンストラクタに渡すだけで使えます。

Worker自体はClassicでもESMでも作成可能で、第2引数で`module`を指定する事でモジュールとなります。今回はWorkerの下流をモジュールとしてimportする構成にしたので、モジュール版でイニシャルします。


```html
<script>    
const worker = new Worker(
    "/sqlite/sqlite3-worker.js",
    { type: "module" }
);

worker.onmessage = event => {
    if (id === "initworker") {
        if (response.success) {
            //Workerのイニシャルが終わってから、DBイニシャルの指示を出す
            worker.postMessage({
               id: "initdb",
               ownerPage: "pagename",
               type: "initdb",
               data: {}
            });
        } else {
            //エラー処理
        }
        return;//抜ける
    } else if(id === "initdb") {
        //DBのイニシャルが終わったら、テーブルイニシャル（初回時用)の支持を出す
        if (response.success) {
            worker.postMessage({
               id: "inittables",
               ownerPage: "pagename",
               type: "inittables",
               data: {}
            });
        } else {
            //エラー処理
        }
        return;//抜ける
    } else if(id === "inittables") {
        if (response.success) {
            //Sqlite3の準備完了
        } else {
            //エラー処理
        }
    }
}
</script>
```

今回はひとつのWorkerで複数のファイルを管理する前提なので、WorkerのイニシャルとDBのイニシャル、テーブルのイニシャルに分けています。

通常のSQLの処理の場合もDBオープン(DBのイニシャル)と、テーブル操作(テーブルのイニシャル)は別なのでそのようにしていますが、
イニシャル処理時にWorkerにパラメータを渡す必要がなければ、すべてまとめてしまうこともできます。

上記コード中のWokerのイニシャルでは、モジュールを非モジュール領域に取り込むことができます。

## 非モジュール領域でモジュールを使う

これは筆者の覚書で少し話がそれますが、モジュールを非モジュール領域で使いたいなら、Windowのプロパティにオブジェクトをセットする方法の他、次のように`import関数とasync`を使った関数でも書くことができます。

ただ軽量なモジュールなら意識しなくていいのですが、読み込みに少し時間のかかるものだとイニシャルを待ち受けないといけません。

次の方法だと、関数呼び出し時の同期が無くなってしまいますが、確実にモジュールオブジェクトが存在する状態でその機能を呼び出せます。


```html
<script>
let mod = null;
const modReady = import("/mod/path.mjs").then(m => {
    mod = m;
});

async function useModFunc() {
   await modReady;
   
   mod.func();
}
</script>
```

同期的(厳密には同期ではないです)に処理をしたいなら、

```html
<script>
let mod = null;

//UIロック
lockScreen();

import("/mod/path.mjs").then(m => {
    mod = m;
    //UIロックの解除
    unlockScreen();
});
</script>
```
とすれば、関数の呼び出しに`async`を不要にできます。

ちなみに、基本的にmodule版のスクリプトはimportをつかって機能を読み込むのですが、中には`<script type="module" src="...">`とする事がwindowに配置されるものもあったりします。

## SQLの実行

メイン側のスクリプトで、Workerのイニシャルが終われば以降は、postMessageメソッドでオブジェクトとして指示を渡します。

```
//メインスクリプト
worker.postMessage({
  id: 1,
  ownerPage: "test",
  type: "find_employee",
  data: {employeeCd: 1234}
});
```
ここで渡すオブジェクトの構造は、Worker定義時の、コードに合わせます。

```
//sqlite-worker.js
self.onmessage = async event => {

    const {
        id,
        ownerPage,
        type,
        data
    } = event.data;
...
```

そして今回は、Wokerの下流で次のようなコードを含んだモジュールが読み込まれている前提です。

```
//Worker サブモジュール
export function execute(db, type, data)
{
    switch (type) {

        case "find_employee":
            return find(db,data);
        default:
            throw new Error(
                `Unknown sample operation: ${type}`
            );
    }
}

function find(db, data)
{
    const rows = [];

    db.exec({
        sql: `
            SELECT
                employee_cd,
                employee_name
            FROM employee
            WHERE employee_cd = ?
        `,
        bind: [
            data.employeeCd
        ],
        rowMode: "object",
        callback: row => {
            rows.push(row);
        }
    });

    return rows;
}
```
