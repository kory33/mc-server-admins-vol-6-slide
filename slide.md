---
marp: true
footer: 'マイクラサーバー管理者の会 vol.6 - 入門　Minecraft プロトコル'
math: mathjax
---
<!-- class: invert -->
<!-- theme: gaia -->
<!-- paginate: true -->

<style>
section {
  font-size: 40px;
}

h3 {
  padding: 0px;
  margin: 0px;
}

h5 {
  font-size: 30px;
  margin: 0px auto;
}

h6 {
  font-size: 25px;
  margin: 0px auto;
}
</style>

<style scoped>
h1 {
  margin: 120px auto;
}

h3 {
  margin: 120px auto;
}
</style>

# **入門　Minecraft プロトコル**

### by @Kory__3 \- at マイクラサーバー管理者の会 vol.6

---

## 自己紹介

- 計算機科学と数学をやっている大学生
  - 普通に「通信に関しては素人」なので、変なこと言ってたら質疑応答とかで突っ込んでください！！！！
- ギガンティック☆整地鯖の運営メンバー
  - ゲームの運営にはそんなに関わっていなくて、主にプラグインの開発、システムアーキテクト、インフラの整備などをやってる

---

<style scoped>
h1 {
  margin: 180px auto;
  justify-content: center;
  text-align: center;
}

h3 {
  font-size: 20px;
  justify-content: right;
  text-align: right;
}
</style>

# Minecraft のオンラインモードで遊んだことはありますか？

---

<style scoped>
h1 {
  margin: 180px auto;
  justify-content: center;
  text-align: center;
}

h3 {
  font-size: 20px;
  justify-content: right;
  text-align: right;
}
</style>

# Minecraft のオンラインモードで遊んだことはありますか？

### あると仮定します

---

## Minecraft のオンラインモードで…

* 一体どのような通信が行われ、クライアントやサーバーがどのような情報をやり取りしてゲームを成り立たせているか、気になったことはありませんか？

* というわけで、今日は Minecraft (Java版) の通信プロトコルについてお話しします

---

<style scoped>
  section > h1 {
    margin: 180px auto;
  }
</style>

# まず、通信に関する一般論から…

---

# 通信に関する一般論

* 通信を行うにあたって、どのような取り決めを行って通信すべきかを先に決める（プロトコルを制定する） 必要がある

* プロトコルを一気に把握したり実装したりするのは厳しいので、一つのプロトコルは抽象度の異なる複数の「レイヤー」に分解される

##### 例えば…

---

###### 通信に関する一般論 >
### プロトコルの階層化

 - Web ブラウザと Web サーバーの通信について考える
 - 最も素朴な階層化では、「何をやり取りしているか」と「どうやり取りしているか」を考えることができる

---

###### 通信に関する一般論 >
### プロトコルの階層化 > Web ブラウザの例

 - Web ブラウザと Web サーバーの通信について考える
 - 最も素朴な階層化では、「何をやり取りしているか」と「どうやり取りしているか」を考えることができる

<br>
まずは「何をやり取りしているか」について考えてみる。

---

###### 通信に関する一般論 >
###### プロトコルの階層化 > Web ブラウザの例 >
### ブラウザは何をやり取りするか？
- ブラウザは、ユーザーがアクセスしたリソースの内容を取得するため、「リクエスト」を送る
- 「リクエスト」に対応して「レスポンス」が得られる。
- 取得したページに「別のリソースを取ってきてくれ」と書かれていたらそれも取りに行く

---

###### 通信に関する一般論 >
###### プロトコルの階層化 > Web ブラウザの例 >
### ブラウザは何をやり取りするか？
- もしユーザーがページに対して何らかのデータを伝えたい場合は、リクエストの中に情報を詰め込んで送る
  - 例えば、Twitterでツイートを投稿する瞬間には、GraphQL API エンドポイントと言われる特定のアドレスにツイート内容を含めた (POST) リクエストが送られる

--- 

###### 通信に関する一般論 >
###### プロトコルの階層化 > Web ブラウザの例 >
### ブラウザは何をやり取りするか？

- 「何をやり取りするか？」の答えは「リクエストを投げてレスポンスを貰う」と言ってよい
- 貰ったレスポンスを元にブラウザがどうすればよいか、といったことは、HTTP といったプロトコルや、HTML、CSS、JPEG などの個々のファイルのルールによって規定されている

--- 

<style scoped>
  section > h2 {
    font-size: 55px;
    margin: 220px auto;
  }
</style>

## 「何をやり取りするか」は分かったので…

---

###### 通信に関する一般論 >
###### プロトコルの階層化 > Web ブラウザの例 >
### 「どうやり取りするか」については？

- データ量を減らすために圧縮されている場合がある
  → GZip 等による圧縮。リクエスト/レスポンスヘッダーにてサーバー/クライアントの合意を取る
- 「一つのリクエスト/レスポンス」といった単位で通信が構造化されている必要がある
  → HTTP (HyperText Transfer Protocol)

---

###### 通信に関する一般論 >
###### プロトコルの階層化 > Web ブラウザの例 >
### 「どうやり取りするか」については？ (続)

- ヘッダー含めてすべて暗号化されている必要があり、サーバー自体の正当性も検証する必要がある (欲張り)
  → TLS (Transport Layer Security)
- 通信中にデータ欠損が起こってはならない + インターネット上のサーバーに繋がる必要がある
  → TCP (Transmission Control Protocol) + IP

---

### 通信に関する一般論 > プロトコルの階層化

「何を」「どのように」やり取りするかという切り分け方でプロトコルスタックを分けたが、もう少し細かい分け方で考えられることも多い。

[TODO: ここに ISO 参照モデルと TCP/IP モデルの画像を入れる]

---

### 通信に関する一般論 > プロトコルの階層化

- ここまでのまとめ
  - 「データの通信」という概念は幾つかの階層に分解することができる
  - 最も素朴には、「何を」「どのように」通信するかという話に二分することができる
  - 例: ウェブサイト閲覧で行われる通信

---

<style scoped>
  section > h1 {
    font-size: 70px;
    margin: 220px auto;
  }
</style>

# Minecraft プロトコルの概要

---

# Minecraft プロトコルの概要

<br>
<br>

やはり「何を通信するか」と「どのように通信するか」に二分して考えることにする

---

###### Minecraft プロトコルの概要 >
### 何を通信するか？

* 「パケット」と言われる単位のデータをクライアントとサーバーが好きなタイミングで相互にやり取りする
  * 単に「パケット」と言ってしまうと「TCP パケット」とすごく紛らわしいので、「Minecraft パケット」と呼び分けることにする

---

###### Minecraft プロトコルの概要 >
### 何を通信するか？

* 「パケット」と言われる単位のデータをクライアントとサーバーが好きなタイミングで相互にやり取りする
  * 単に「パケット」と言ってしまうと「TCP パケット」とすごく紛らわしいので、「Minecraft パケット」と呼び分けることにする

---

###### Minecraft プロトコルの概要 > 何を通信するか？
### Minecraft パケットとは？

* サーバーやクライアントが、Minecraft セッション上でやり取りする情報の塊
* 複数種類がある。例えば…
  * 「プレーヤーがサーバーに自分の位置を報告する」
    * [`Set Player Position`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Player_Position)
  * 「サーバーが特定プレーヤーの体力を書き換える」
    * [`Set Health`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Health)

---

###### Minecraft プロトコルの概要 > 何を通信するか？
### Minecraft パケットとは？

* サーバーやクライアントが、Minecraft セッション上でやり取りする情報の塊
* 複数種類がある。例えば…
  * 「プレーヤーがサーバーに自分の位置を報告する」
    * [`Set Player Position`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Player_Position)
  * 「サーバーが特定プレーヤーの体力を書き換える」
    * [`Set Health`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Health)

---

###### Minecraft プロトコルの概要 > 何を通信するか？
### Minecraft パケットとは？

* サーバー向け (Serverbound) と クライアント向け (Clientbound) のものがある。例えば…
  * 「プレーヤーがサーバーに自分の位置を報告する」[`Set Player Position`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Player_Position) は Serverbound
  * 「サーバーが特定プレーヤーの体力を書き換える」[`Set Health`](https://wiki.vg/index.php?title=Protocol&oldid=17749#Set_Health) は Clientbound

---

###### Minecraft プロトコルの概要
### どのように通信するか？

<br>
<br>

次に、「どのように通信するか？」について見ていく

---

###### Minecraft プロトコルの概要
### どのように通信するか？

* 通信内容が暗号化されていて欲しい
  * 1024-bit RSA での鍵交換 + AES/CFB8 共通鍵ストリーム暗号が使える
* データ通信量が多く、単調なデータ(地形データなど)が多い
  * Java の `java.util.zip` がサポートしている zlib 圧縮が使える

---

###### Minecraft プロトコルの概要 >
### どのように通信するか？

* 相互に好きなタイミングでデータを送ることができ、データ欠損が起こってはならない
  * TCP + IP

<br>

つまり、暗号化と zlib 圧縮を TCP コネクション上で利用する

---

###### Minecraft プロトコルの概要

まとめると…

<span style="color: #ff6">TCP (+ AES/CFB8 ストリーム暗号利用可能) (+ zlib 圧縮利用可能) によって確立された双方向通信路</span>で、<span style="color: #f8e">「(Minecraft)パケット」と言われる単位のデータをクライアントとサーバーが好きなタイミングで相互にやり取り</span>する

(残りの時間では、主にピンクの部分についてお話しします)

---

## Minecraft プロトコルの詳細

- https://wiki.vg/Protocol というサイトで情熱的な有志がプロトコルを解析して公開している
  - (今日の発表は主にここを参考にしています)
  - 以下、基本的に Minecraft 1.19 についての内容をお話しします

---

##### Minecraft プロトコルの詳細 >
### パケットのフォーマット

- パケットは以下の三つのデータを持つ
  - パケット長 (整数; `VarInt`)
  - パケットID (整数; `VarInt`)
  - パケットデータ (バイト列)

---

##### Minecraft プロトコルの詳細 >
### パケットのフォーマット

パケット ID は、当該 Minecraft パケットがどのような意味に解釈されるべきかを指定する

- ここで言う「意味」は、例えば、プレーヤーの座標同期指示 ([Synchronize Player Position](https://wiki.vg/index.php?title=Protocol&oldid=17749#Synchronize_Player_Position)) なのか、エンティティをスポーンさせる指示 ([Spawn Entity](https://wiki.vg/index.php?title=Protocol&oldid=17749#Spawn_Entity)) なのか、など

---

##### Minecraft プロトコルの詳細 >
### パケットのフォーマット

- パケットの意味は、パケットデータを「フィールド」の列に解釈する方法を定める

  - 例えば、プレーヤーの座標同期指示 ([Synchronize Player Position](https://wiki.vg/index.php?title=Protocol&oldid=17749#Synchronize_Player_Position)) のパケットならば、プレーヤーの座標、視点方向、値が相対的かどうかのフラグ、Teleport ID (後述)、乗り物から降りた状態でいるかどうかの指定、といったフィールドを持つ

---

##### Minecraft プロトコルの詳細 > パケットのフォーマット
### オプショナルフィールドについて

* Minecraft パケットは、すべてのフィールドが必須となっているわけではない。「他のフィールドの値に依存して、値があるかどうかが決まる」フィールドがある。

* [wiki.vg](https://wiki.vg/index.php?title=Protocol&oldid=17749) ではこれを `Optional field` と呼んでいる。

* 例:
  - [Stop Sound](https://wiki.vg/index.php?title=Protocol&oldid=16907#Stop_Sound) パケット (図を描く時間が無かったので拙著の [Qiita記事](https://qiita.com/Kory__3/items/250c1b00d0e04813c45d#%E9%81%94%E6%88%90%E3%81%97%E3%81%9F%E3%81%84%E5%A4%89%E6%8F%9B) を参考ください…)

---

##### Minecraft プロトコルの詳細 >
### パケットのやり取りの例：キープアライブ

-  クライアントは、キープアライブ (Keep alive) パケットに対して返事をする必要がある
    - 「まだ接続してますよ」という確証をサーバーに与える
 
---

##### Minecraft プロトコルの詳細 >
### パケットのやり取りの例：キープアライブ (続)

- [Keep Alive (Clientbound)](https://wiki.vg/index.php?title=Protocol&oldid=17749#Keep_Alive_.28clientbound.29) がランダムな `Keep Alive ID` とともに送られてくる
- クライアントは、送られてきた `Keep Alive ID` を 30 秒以内に [Keep Alive (Serverbound)](https://wiki.vg/index.php?title=Protocol&oldid=17749#Keep_Alive_.28serverbound.29) パケットにて送り返さねばならない
- クライアントが 30 秒返事をしなかった場合、サーバーは [Disconnect](https://wiki.vg/index.php?title=Protocol&oldid=17749#Disconnect_.28play.29) パケットを送出し、TCPコネクションを切断する

---

##### Minecraft プロトコルの詳細 >
### バージョン依存性について

- Minecraft プロトコルには「プロトコルバージョン」がある
  - 1.19.1 はプロトコルバージョン 760
- プロトコルバージョンは、 Minecraft のパッチバージョンを跨ぐときにも切り替わることがある

---

##### Minecraft プロトコルの詳細 >
### バージョン依存性について (続)

- プロトコルバージョンが上がると、パケットIDとパケットのマッピングが一新されたり、フィールドの型が変わったりする
  - プロトコルが非互換になる
  - バージョンを大きく跨いでサーバーに参加できないのはこのため

---

##### Minecraft プロトコルの詳細 >
### バージョン依存性について (注)

- ただし、ハンドシェイクと参加前の ping に関するパケットだけは 1.7.10 から (少なくとも) 1.17.1 まで非互換変更がない
  - マイクラのサーバー選択画面で「サーバーのバージョンが新しす/古すぎます」といった表示ができているのは、ログイン周りのパケットが複数プロトコルバージョンを跨いで変わっていないことに依存している

---

<style scoped>
  section > h2 {
    font-size: 55px;
    margin: 220px auto;
  }
</style>

## Minecraft プロトコルの知識があれば何ができるか？

---

##### Minecraft プロトコルの知識があれば何ができるか？ >
### BungeeCord

- 部分的なプロトコルの知識が必要
- 「プレーヤーがサーバー S1 から S2 に移動したとき、S1 に Disconnect パケットを送りつつ、プレーヤーには [Respawn](https://wiki.vg/index.php?title=Protocol&oldid=17749#Respawn) パケットを送り、S2 へのログイン処理を肩代わりし、S2 の地形データをクライアントに受け流す」

---

##### Minecraft プロトコルの知識があれば何ができるか？ >
### ProtocolLib

- Spigot サーバー (Minecraft 互換サーバーであって、プラグイン機構、プラグイン向けAPIとその実装が追加されているもの) 上で動くプラグインで、次の機能を他のプラグインに提供する：
  - Spigot の標準 API では不可能な操作をパケットを通して行えるようにする
  - パケット操作をある程度抽象化する

---

##### Minecraft プロトコルの知識があれば何ができるか？ >
### ViaVersion / ViaBackwards

- 古いクライアントを新しいバージョンのMinecraftサーバーに接続 (ViaVersion) させたり、その逆 (ViaBackwards) を行うプラグイン達
  - それぞれのプロトコルの差分を吸収して、差分を合成することで実現している（すごい！）

---

##### Minecraft プロトコルの知識があれば何ができるか？ >
### E2Eテスト

- 「BungeeCord で束ねられたサーバー群がユーザーから見て正しく動作するか」を自動で検証したいことがある
  - 主に Spigot プラグイン開発者とか、 BungeeCord プラグイン開発者とか、データパック開発者など
  - サーバー管理者も、一定のプラグインの組み合わせが正しく動くか、などを自動テスト出来て嬉しいかもしれない

---

## 参考
- [Protocol - wiki.vg](https://wiki.vg/Protocol)
- [Scala 3 のマクロを使って Minecraft のプロトコルデコーダを自動導出する - Qiita](https://qiita.com/Kory__3/items/250c1b00d0e04813c45d)
- [インターネット・プロトコル・スイート - Wikipedia](https://ja.wikipedia.org/wiki/%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%BC%E3%83%8D%E3%83%83%E3%83%88%E3%83%BB%E3%83%97%E3%83%AD%E3%83%88%E3%82%B3%E3%83%AB%E3%83%BB%E3%82%B9%E3%82%A4%E3%83%BC%E3%83%88)
- [ProtocolLib | SpigotMC](https://www.spigotmc.org/resources/protocollib.1997/)
- [ViaVersion | SpigotMC](https://www.spigotmc.org/resources/viaversion.19254/)

---

## 参考

- [ViaVersion's `protocols` package](https://github.com/ViaVersion/ViaVersion/tree/ce4f21b7d82446a5632c4c16de031f62f3209248/common/src/main/java/com/viaversion/viaversion/protocols)
- [ViaBackwards | SpigotMC](https://www.spigotmc.org/resources/viabackwards.27448/)

