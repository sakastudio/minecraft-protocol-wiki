# Server List Ping
https://wiki.vg/Server_List_Ping \
**Server List Ping(SLP)** は、Minecraftサーバーが提供するインターフェースで、通常のポート経由でMOTD、プレイヤー数、最大プレイヤー数、サーバーのバージョンの問い合わせを行います。 
SLPは [プロトコル](https://wiki.vg/Protocol) であり、[クエリ](https://wiki.vg/Query) ではないため、このインターフェースは常時稼働します。 
Notchainクライアントはこのインターフェースを使用してサーバーリストを表示します。
SLPの手順はMinecraft ver1.7で下位互換がない形式で変更されましたが、現在のサーバーは[新しいプロセス](https://c4k3.github.io/wiki.vg/Server_List_Ping.html#Current)と[古いプロセス](https://c4k3.github.io/wiki.vg/Server_List_Ping.html#1.6)の両方をサポートしています。

## 現行バージョン

これは通常のクライアント-サーバー [プロトコル](https://wiki.vg/Protocol) を使用します。一般的なフォーマットについてはその記事を参照してください。

**ハンドシェイク**

最初に、クライアントはステータスを1に設定した [ハンドシェイク](https://c4k3.github.io/wiki.vg/Protocol.html#Handshake) パケットを送信します。


| パケットID | フィールド名 | フィールドタイプ | 詳細 |
-- | -- | -- |-- 
|0x00| プロトコルバージョン  | VarInt | [プロトコルバージョン](https://wiki.vg/Protocol_version_numbers) を参照してください。クライアントがサーバーに接続する際に使用するバージョンの確認をします(ping自体は重要ではありません)。クライアントがどのバージョンを使用するか確認するためにpingをする場合、慣習的に-1に設定する必要があります。 |
|0x00| サーバーアドレス| String | 接続に使用されたホスト名またはIPアドレスです(例:localhostまたは127.0.0.1)。Notchianサーバはこの情報を使用しません。SRVレコードが定義されている場合はリダイレクトされます。_minecraft._tcp.example.comがmc.example.orgを指している場合、example.comに接続するユーザーはmc.example.orgにリダイレクトします。|
|0x00| サーバーポート | Unsigned Short | デフォルトは25565です。Notchianサーバーはこの情報を使用しません。|
|0x00| 次のステータス | VarInt  | [ステータス](https://wiki.vg/Protocol#Status) の場合は1に設定しますが、[ログイン](https://wiki.vg/Protocol#Login) の場合は2にすることもできます。|


**リクエスト**

クライアントは [リクエスト](https://wiki.vg/Protocol#Request) パケットで追跡をします。これはフィールドを持ちません。

| パケットID | フィールド名 | フィールドタイプ | 詳細 |
-- | -- | -- |-- 
|0x00||||

**レスポンス**

サーバーはレスポンスパケットで応答すべきです。なお、理由は不明ですが、Notchianサーバーは [ping](https://wiki.vg/Protocol#Ping) パケットを受信したら30秒待機し、タイムアウトして応答パケットを送信します。

| パケットID | フィールド名 | フィールドタイプ | 詳細 |
-- | -- | -- |-- 
|0x00|JSONレスポンス|String |他のStringと同様に、Varintで表現された長さが前につきます。|

JSONレスポンスのフォーマットは以下の通りです。

```
{
    "version": {
        "name": "1.8.7",
        "protocol": 47
    },
    "players": {
        "max": 100,
        "online": 5,
        "sample": [
            {
                "name": "thinkofdeath",
                "id": "4566e69f-c907-48ee-8d71-d7ba5aa00d20"
            }
        ]
    },
    "description": {
        "text": "Hello world"
    },
    "favicon": "data:image/png;base64,<data>"
}
```

descriptionフィールドは [チャット](https://wiki.vg/Chat) オブジェクトです。
Notchianサーバーは実際のチャットコンポーネントデータを提供する手段を持っていないことに注意してください。代わりに、セクション記号ベースのコードがオブジェクトのテキスト内に埋め込まれます。


faviconフィールドはオプションです。sampleフィールドは設定しなければなりませんが、空でもよいです。


faviconはBase64エンコードされたPNG画像で、先頭にdata:image/png;base64, を付加したものです。

レスポンスパケットを受信した後、クライアントは次のパケットを送信してサーバーのレイテンシーを計算するのに役立ててもよいし、それが必要ない場合はそこで切断しても良い。
クライアントが適切にフォーマットされた応答を受け取らなかった場合は、代わりに (過去バージョンのping)[https://wiki.vg/Server_List_Ping#1.6] を試みます。

**Ping**

このまま処理を続けると、クライアントは重要ではない何らかのペイロードを含む[Pingパケット](https://wiki.vg/Protocol#Ping)を送信することになります。


| パケットID | フィールド名 | フィールドタイプ | 詳細 |
-- | -- | -- |-- 
|0x01|ペイロード|Long|任意の数値を指定します。Notchianクライアントは、ミリ秒単位でカウントされるシステム依存の時間値を使用します。|


**Pong**

サーバーは[Pongパケット](https://wiki.vg/Protocol#Pong)で応答した後、接続を終了します。


| パケットID | フィールド名 | フィールドタイプ | 詳細 |
-- | -- | -- |-- 
0x01|ペイロード|Long|クライアントから送られてきたものと同じであること|

**LAN経由のping (シングルプレイ時にLANで開く)**

Singeplayerには "Open to LAN "という機能があります。Minecraft は (サーバーリストで) UDP ポートをバインドし、224.0.2.60:4445 (そう、これは実際の IP で、あなたがどのネットワークにいるか、ローカル IP アドレスが何であるかは関係ありません) への接続を待ちます。LANに開く」をクリックすると、Minecraftは1.5秒ごとにパケットをこのアドレスに送信します。[MOTD]{motd}[/MOTD][AD]{port}[/AD]. Minecraft は以下の文字列をチェックしているようです。motd]、[/motd]、[ad]、[/ad]です。基本的に他のものは重要ではありません。タグの前(と後または間)にデタラメを書くこともできます。カラーコードはサポートされていないようです（ToDo: Check again. ここでは動作しませんでした）。AD]と[/AD]の間は、サーバーのポートです。数字でない場合は、25565が使われます。範囲外の場合、接続しようとするとエラーが表示されます。IPアドレスは、送信者のものと同じです。文字列は、UTF-8でエンコードされています。

サーバーサイドで実装する場合は、224.0.2.60:4445にテキスト（ペイロード）を含むパケットを送信すればOKです。クライアント側で実装する場合は、UDPソケットをバインドして接続を待ちます。これにはMulticastSocketが使えます。

**サンプルコード**

[C#](https://gist.github.com/csh/2480d14fbbb33b4bbae3)\
[Java](https://gist.github.com/zh32/7190955)\
[Python](https://gist.github.com/1209061)\
[Python3](https://gist.github.com/ewized/97814f57ac85af7128bf)\
[PHP](https://github.com/xPaw/PHP-Minecraft-Query)\
[LAN Server Listener (Java)](https://gitlab.bixilon.de/bixilon/minosoft/-/blob/development/src/main/java/de/bixilon/minosoft/protocol/protocol/LANServerListener.java)
