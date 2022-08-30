# 入門　Minecraft プロトコル

## 大体の流れ

 - 軽い自己紹介
   - 計算機科学と数学をやっている大学生
     - 実は「通信に関してはド素人」なので、変なこと言ってたら質疑応答とかで突っ込んでください！！！！
   - ギガンティック☆整地鯖の運営メンバー
     - ゲームの運営にはそんなに関わっていなくて、主にプラグインの開発、システムアーキテクト、インフラの整備などをやってる

 - 導入
   - Minecraft のオンラインモードで遊んだことはありますよね？
   - 一体どのような通信が行われ、クライアントやサーバーがどのような情報をやり取りしてゲームを成り立たせているか、気になったことはありませんか？
   - というわけで、今日は Minecraft (Java版) の通信プロトコルについてお話しします

 - 事前知識の紹介
   - インターネットにおける階層化された通信概念の紹介
     - 「何をやり取りしているか」と「どうやり取りしているか」の二つを気にするとわかりやすい
       - 普段使っている HTTPS によるwebブラウジングの例を出す
         - 「何をやり取りしているか」？
           - ブラウザが欲しいものをサーバーに取りに行く
             - (サーバーから渡されたデータに「これも取りに来て」と書かれていたらそれも取りに行く感じです)
           - ヘッダー情報によるブラウザとサーバーの (一回きりの) 相互コミュニケーション
           - Web ページの HTML ドキュメント、画像、JavaScript、CSS 等のアセットが送られてくる
           - ブラウザがウェブサイトに変更を及ぼしたいときは、リクエストに情報を詰め込む
             - 「POST リクエスト」と言われる作法にてこれが行われる
         - 「どうやり取りしているか」？
           - データ量を減らすために圧縮されている場合がある
             - GZip 等による圧縮。有無はヘッダーにて指定
           - 「一つのリクエスト」といった単位で構造化されている必要があり、ヘッダー含めてすべて暗号化されている必要があり、サーバー自体の正当性も検証する必要がある (欲張り…)
             - TLS (Transport Layer Security)
           - データ欠損が起こってはならない
             - TCP (Transmission Control Protocol)
           - 世界中のサーバーにアクセスする必要がある
             - IP (Internet Protocol)
           - 通信路の確保 (PPP / Ethernet / ...)
         - 俯瞰図
           - (あとでこの図の「Minecraftバージョン」をまた出して、この図と並べるので、見比べてみてください。)
     - 上のような「階層」をより細かく分類したモデルがいくつかある
       - （名前と絵だけ出して軽く説明する）
         - OSI 参照モデル
         - Internet protocol suite (a.k.a. TCP/IP)

 - ここまでのまとめ
   - 「データの通信」という概念は幾つかの階層に分解することができる
   - 最も素朴には、「何を」「どのように」通信するかという話に二分することができる
   - 例: ウェブサイト閲覧で行われる通信

 - Minecraft プロトコルの概要
   - 何を通信している？
     - 「パケット」と言われる単位のデータをクライアントとサーバーが好きなタイミングで相互にやり取りする
     - 「TCP パケット」とすごく紛らわしいので、「Minecraft パケット」と呼び分けることにする
     - 「Minecraft パケット」とは？
       - サーバーやクライアントが、Minecraft セッション上でやり取りする情報の塊
       - 複数種類がある
         - 例:
           - 「プレーヤーがサーバーに自分の位置を報告する ([`Set Player Position`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Player_Position))」
           - 「サーバーが特定プレーヤーの体力を書き換える ([`Set Health`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Health))」
       - サーバー向け (Serverbound) と クライアント向け (Clientbound) のものがある
         - 例:
           - [`Set Player Position`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Player_Position) は Serverbound
           - [`Set Health`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Health) は Clientbound

   - どのように通信している？ 
     - 相互に好きなタイミングでデータを送ることができ、データ欠損が起こってはならない
       - TCP (Transmission Control Protocol)
     - 世界中のサーバーにアクセスする必要がある
       - IP (Internet Protocol)

   - まとめると…
     - TCP (+ オプショナルで AES/CFB8 ストリーム暗号付き) (+ オプショナルで zlib 圧縮付き) によって確立された双方向通信路で、「(Minecraft)パケット」と言われる単位のデータをクライアントとサーバーが好きなタイミングで相互にやり取りする
       - (これは一気にしゃべり過ぎなので、適当に色を付けるなどして分解して提示する)

 - Minecraft プロトコルの詳細
   - https://wiki.vg/Protocol というサイトで情熱的な有志がプロトコルを解析して公開している
     - (今日の発表は主にここを参考にしています)
     - 以下、基本的に Minecraft 1.19 についての内容をお話しします

   - Minecraft パケットのフォーマット
     - [参考 - wiki.vg](https://wiki.vg/index.php?title=Protocol&oldid=17749#Packet_format)
     - パケットIDとパケットフィールド
       - (図で)
         - パケット長(整数; `VarInt`)とパケットID(整数; `VarInt`)とパケットデータ(バイト列)の三つ組
         - パケット ID は、当該 Minecraft パケットがどのような意味に解釈されるべきかを指定する
           - ここで言う「意味」は、例えば、プレーヤーの座標同期指示 ([Synchronize Player Position](https://wiki.vg/index.php?title=Protocol&oldid=17749#Synchronize_Player_Position)) なのか、エンティティをスポーンさせる指示 ([Spawn Entity](https://wiki.vg/index.php?title=Protocol&oldid=17749#Spawn_Entity)) なのか、など
         - パケットの意味は、パケットデータを「フィールド」の列に解釈する方法を定める
           - フィールドはパケットの
           - 例えば、プレーヤーの座標同期指示 ([Synchronize Player Position](https://wiki.vg/index.php?title=Protocol&oldid=17749#Synchronize_Player_Position)) のパケットならば、プレーヤーの座標、視点方向、値が相対的かどうかのフラグ、Teleport ID (後述)、乗り物から降りた状態でいるかどうかの指定、といったフィールドを持つ
     - オプショナルフィールド
       - Minecraft パケットは、すべてのフィールドが必須となっているわけではない
       - 「他のフィールドの値に依存して、値があるかどうかが決まる」フィールドがある
       - [wiki.vg](https://wiki.vg/index.php?title=Protocol&oldid=17749) ではこれを `Optional`
       - 例:
         - [Stop Sound](https://wiki.vg/index.php?title=Protocol&oldid=16907#Stop_Sound) パケットを考える
         - (TODO: 表を挿入して口頭で説明する)

   - 例
     - ハートビート
       - クライアントは、「まだ接続してますよ」という確証をサーバーに与えるために、キープアライブ (Keep alive) パケットと呼ばれる、サーバーから定期的に送られてくるパケットに対して返事をする必要がある (もし返事を忘れたりするとサーバーによって強制切断される)
         - TODO: 具体的なパケット名を出して説明する

     - エンティティ (プレーヤー含む) の位置情報と速度情報のやり取り
       - プレーヤーはサーバーに位置を報告する
         - TODO: 具体的なパケット名を出して説明する

   - バージョン依存性
     - Minecraft プロトコルには「プロトコルバージョン」がある
       - 1.19.1 はプロトコルバージョン 760
     - プロトコルバージョンは、 Minecraft のパッチバージョンを跨ぐときにも切り替わることがある
     - プロトコルバージョンが上がると、パケットIDとパケットのマッピングが一新されたり、フィールドの型が変わったりする
       - プロトコルが非互換になる
       - バージョンを大きく跨いでサーバーに参加できないのはこのため
     - ただし、ハンドシェイクと参加前の ping に関するパケットだけは 1.7.10 から (少なくとも) 1.17.1 まで非互換変更がない
       - マイクラのサーバー選択画面で「サーバーのバージョンが新しす/古すぎます」といった表示ができているのは、ログイン周りのパケットが複数プロトコルバージョンを跨いで変わっていないことに依存している

 - Minecraft プロトコルの知識があれば何ができるか？
   - BungeeCord
     - 部分的なプロトコルの知識が必要
     - (シーケンス図で)「プレーヤーがサーバー S1 から S2 に移動したとき、S1 に Disconnect パケットを送りつつ、プレーヤーには change world パケットを送り、S2 のログイン処理を肩代わりし、S2 の地形データをクライアントに受け流す」

   - ProtocolLib
     - Spigot サーバー (Minecraft 互換サーバーであって、プラグイン機構、プラグイン向けAPIとその実装が追加されているもの) 上で動くプラグインで、次の機能を他のプラグインに提供する：
       - Spigot の標準 API では不可能な操作をパケットを通して行えるようにする
       - パケット操作をある程度抽象化する
       - (全 Minecraft バージョンに対して継続的にメンテナンスするのは先述の「バージョン依存性」によってコストがかさむので、)基本的に ProtocolLib の最新版は Spigot (Minecraftサーバー) の最新版しかサポートしていない
     - 良いところ
       - 人力でパケットIDとパケットデータのバイナリを人力で書くみたいな苦行をしないで良くなる
       - パケットの実体 -> 対応するパケットID の解決を受け持ってくれるため、パケット操作を要するプラグインが複数バージョンで動くようになる（ことがある）
     - つらいところ
       - リフレクションでパケットIDを持ってきたりしており、コンパイル時にパケットの妥当性の検査が行われない
         - プラグインのちゃんとしたデバッグが必要…

   - ViaVersion / ViaBackwards
     - 古いクライアントを新しいバージョンのMinecraftサーバーに接続 (ViaVersion) させたり、その逆 (ViaBackwards) を行うプラグイン達
       - それぞれのプロトコルの差分を吸収して、差分を合成することで実現している
         - 例えば、クライアントが 1.11 でサーバーが 1.12 の場合、
           - (図で)
             - Serverbound パケットは 1.11 -> 1.11.1 -> 1.12
             - Clientbound パケットは 1.11 <- 1.11.1 <- 1.12
             - のような変換スタックを通すことでパケットの翻訳が行える

   - E2Eテスト
     - 「BungeeCord で束ねられたサーバー群がユーザーから見て正しく動作するか」を自動で検証したいことがある
       - 主に Spigot プラグイン開発者とか、 BungeeCord プラグイン開発者とか、データパック開発者など
       - サーバー管理者も、一定のプラグインの組み合わせが正しく動くか、などを自動テスト出来て嬉しいかもしれない
       - (E2E テストは記述コストが重くなりがちなので、プレーヤーを必要とせずに自動で発火ができる処理のテストなどは、Spigot プラグインなどでやってしまったほうが楽かもしれない)
     - クライアント視点での動作が自動で確かめられるのは結構嬉しそう
     - エミュレータを自前で実装したり、クライアントからの/への通信を傍受して内容を検証したい場合にはプロトコルの知識が要る

 - 参考
   - [Protocol - wiki.vg](https://wiki.vg/Protocol)
   - [Scala 3 のマクロを使って Minecraft のプロトコルデコーダを自動導出する - Qiita](https://qiita.com/Kory__3/items/250c1b00d0e04813c45d)
   - [インターネット・プロトコル・スイート - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E3%83%BB%E3%83%97%E3%83%AD%E3%83%88%E3%82%B3%E3%83%AB%E3%83%BB%E3%82%B9%E3%82%A4%E3%83%BC%E3%83%88)
   - [ProtocolLib | SpigotMC](https://www.spigotmc.org/resources/protocollib.1997/)
   - [ViaVersion | SpigotMC](https://www.spigotmc.org/resources/viaversion.19254/)
   - [ViaVersion's `protocols` package](https://github.com/ViaVersion/ViaVersion/tree/ce4f21b7d82446a5632c4c16de031f62f3209248/common/src/main/java/com/viaversion/viaversion/protocols)
   - [ViaBackwards | SpigotMC](https://www.spigotmc.org/resources/viabackwards.27448/)
