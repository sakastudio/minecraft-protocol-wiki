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


**パケットID 0x00**
| フィールド名 | フィールドタイプ | 詳細 |
-- | --| --- 
| プロトコルバージョン  | VarInt | [プロトコルバージョン](https://wiki.vg/Protocol_version_numbers) を参照してください。クライアントがサーバーに接続する際に使用するバージョンの確認をします(ping自体は重要ではありません)。クライアントがどのバージョンを使用するか確認するためにpingをする場合、慣習的に-1に設定する必要があります。 |
| サーバーアドレス| String | 接続に使用されたホスト名またはIPアドレスです(例:localhostまたは127.0.0.1)。Notchianサーバはこの情報を使用しません。SRVレコードが定義されている場合はリダイレクトされます。_minecraft._tcp.example.comがmc.example.orgを指している場合、example.comに接続するユーザーはmc.example.orgにリダイレクトします。|
| サーバーポート | Unsigned Short | デフォルトは25565です。Notchianサーバーはこの情報を使用しません。|
| 次のステータス | VarInt  | [ステータス](https://wiki.vg/Protocol#Status) の場合は1に設定しますが、[ログイン](https://wiki.vg/Protocol#Login) の場合は2にすることもできます。|


=== Request ===

The client follows up with a [[Protocol#Request|Request]] packet. This packet has no fields.

{| class="wikitable"
! Packet ID
! Field Name
! Field Type
! Notes
|-
| 0x00
|colspan="3"| ''no fields''
|}

=== Response ===

The server should respond with a [[Protocol#Response|Response]] packet. Note that Notchian servers will for unknown reasons wait to receive the following [[Protocol#Ping|Ping]] packet for 30 seconds before timing out and sending Response.

{| class="wikitable"
! Packet ID
! Field Name
! Field Type
! Notes
|-
| 0x00
| JSON Response
| String
| See below; as with all strings this is prefixed by its length as a VarInt
|}

The JSON Response field is a [[wikipedia:JSON|JSON]] object which has the following format:

<syntaxhighlight lang="javascript">
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
</syntaxhighlight>

The ''description'' field is a [[Chat]] object.  Note that the Notchian server has no way of providing actual chat component data; instead section sign-based codes are embedded within the text of the object.

The ''favicon'' field is optional. The ''sample'' field must be set, but can be empty.

The ''favicon'' should be a [[wikipedia:Portable Network Graphics|PNG]] image that is [[wikipedia:Base64|Base64]] encoded (without newlines: <code>\n</code>, new lines no longer work since 1.13) and prepended with <code>data:image/png;base64,</code>.

After receiving the Response packet, the client may send the next packet to help calculate the server's latency, or if it is only interested in the above information it can disconnect here.

If the client does not receive a properly formatted response, then it will instead attempt a [[Server_List_Ping#1.6|legacy ping]].

=== Ping ===

If the process is continued, the client will now send a [[Protocol#Ping|Ping]] packet containing some payload which is not important.

{| class="wikitable"
! Packet ID
! Field Name
! Field Type
! Notes
|-
| 0x01
| Payload
| Long
| May be any number. Notchian clients use a system-dependent time value which is counted in milliseconds.
|}

=== Pong ===

The server will respond with the [[Protocol#Pong|Pong]] packet and then close the connection.

{| class="wikitable"
! Packet ID
! Field Name
! Field Type
! Notes
|-
| 0x01
| Payload
| Long
| Should be the same as sent by the client
|}

=== Ping via LAN (Open to LAN in Singleplayer) ===

In Singeplayer there is a function called "Open to LAN". Minecraft (in the serverlist) binds a UDP port and listens for connections to <code>224.0.2.60:4445</code> (Yes, that is the actual IP, no matter in what network you are or what your local IP Address is). If you click on "Open to LAN" Minecraft sends a packet every 1.5 seconds to the address: <code>[MOTD]{motd}[/MOTD][AD]{port}[/AD]</code>. Minecraft seems to check for the following Strings: <code>[MOTD]</code>, <code>[/MOTD]</code>, <code>[AD]</code>, <code>[/AD]</code>. Basically everything else is not important, you can also write bullshit before the tags (and after or between). Color codes seems to be unsupported (ToDo: Check again. Did not work here). Between <code>[AD]</code> and <code>[/AD]</code> is the servers port. If it is not numeric, 25565 will be used. If it is out of range, an error is being displayed when trying to connect. The IP Address is the same as the senders one. The string is encoded with UTF-8.

To implement it server side, just send a packet with the text (payload) to <code>224.0.2.60:4445</code>. If you are client side, bind a UDP socket and listen for connections. You can use a <code>MulticastSocket</code> for that.

=== Examples ===

* [https://gist.github.com/csh/2480d14fbbb33b4bbae3 C#]
* [https://gist.github.com/zh32/7190955 Java]
* [https://gist.github.com/1209061 Python]
* [https://gist.github.com/ewized/97814f57ac85af7128bf Python3]
* [https://github.com/xPaw/PHP-Minecraft-Query PHP]
* [https://gitlab.bixilon.de/bixilon/minosoft/-/blob/development/src/main/java/de/bixilon/minosoft/protocol/protocol/LANServerListener.java LAN Server Listener (Java)]

== 1.6 ==

This uses a protocol which is compatible with the client-server protocol as it was before the Netty rewrite. Modern servers recognize this protocol by the starting byte of <code>fe</code> instead of the usual <code>00</code>.

=== Client to server ===

The client initiates a TCP connection to the server on the standard port. Instead of doing auth and logging in (as detailed in [[Protocol]] and [[Protocol Encryption]]), it sends the following data, expressed in hexadecimal:

# <code>FE</code> — packet identifier for a server list ping
# <code>01</code> — server list ping's payload (always 1)
# <code>FA</code> — packet identifier for a plugin message
# <code>00 0B</code> — length of following string, in characters, as a short (always 11)
# <code>00 4D 00 43 00 7C 00 50 00 69 00 6E 00 67 00 48 00 6F 00 73 00 74</code> — the string <code>MC|PingHost</code> encoded as a [http://en.wikipedia.org/wiki/UTF-16 UTF-16BE] string
# <code>XX XX</code> — length of the rest of the data, as a short. Compute as <code>7 + len(hostname)</code>, where <code>len(hostname)</code> is the number of bytes in the UTF-16BE encoded hostname.
# <code>XX</code> — [[Protocol version numbers#Versions before the Netty rewrite|protocol version]], e.g. <code>4a</code> for the last version (74)
# <code>XX XX</code> — length of following string, in characters, as a short
# <code>...</code> — hostname the client is connecting to, encoded as a [http://en.wikipedia.org/wiki/UTF-16 UTF-16BE] string
# <code>XX XX XX XX</code> — port the client is connecting to, as an int.

All data types are big-endian.

Example packet dump:

0000000: fe01 fa00 0b00 4d00 4300 7c00 5000 6900  ......M.C.|.P.i.
0000010: 6e00 6700 4800 6f00 7300 7400 1949 0009  n.g.H.o.s.t..I..
0000020: 006c 006f 0063 0061 006c 0068 006f 0073  .l.o.c.a.l.h.o.s
0000030: 0074 0000 63dd                           .t..c.

=== Server to client ===

The server responds with a 0xFF kick packet. The packet begins with a single byte identifier <code>ff</code>, then a two-byte big endian short giving the length of the following string in characters. You can actually ignore the length because the server closes the connection after the response is sent.

After the first 3 bytes, the packet is a UTF-16BE string. It begins with two characters: <code>§1</code>, followed by a null character. On the wire these look like <code>00 a7 00 31 00 00</code>.

The remainder is null character (that is <code>00 00</code>) delimited fields:

# Protocol version (e.g. <code>74</code>)
# Minecraft server version (e.g. <code>1.8.7</code>)
# Message of the day (e.g. <code>A Minecraft Server</code>)
# Current player count
# Max players

The entire packet looks something like this:

                 <---> first character
0000000: ff00 2300 a700 3100 0000 3400 3700 0000  ....§.1...4.7...
0000010: 3100 2e00 3400 2e00 3200 0000 4100 2000  1...4...2...A. .
0000020: 4d00 6900 6e00 6500 6300 7200 6100 6600  M.i.n.e.c.r.a.f.
0000030: 7400 2000 5300 6500 7200 7600 6500 7200  t. .S.e.r.v.e.r.
0000040: 0000 3000 0000 3200 30                   ..0...2.0

'''Note: ''' When using this protocol with servers on version 1.7.x and above, the protocol version (first field) in the response will always be <code>127</code> which is not a real protocol number, so older clients will always consider this server incompatible.

=== Examples ===

* [https://gist.github.com/6281388 Ruby]
* [https://github.com/winny-/mcstat PHP]

== 1.4 to 1.5 ==

Prior to the Minecraft 1.6, the client to server operation is much simpler, and only sends <code>FE 01</code>, with none of the following data.

=== Examples ===

* [https://gist.github.com/5795159 PHP]
* [https://gist.github.com/4574114 Java]
* [https://gist.github.com/6223787 C#]

== Beta 1.8 to 1.3 ==

Prior to Minecraft 1.4, the client only sends <code>FE</code>.

Additionally, the response from the server only contains 3 fields delimited by <code>§</code>:

# Message of the day (e.g. <code>A Minecraft Server</code>)
# Current player count
# Max players

The entire packet looks something like this:

                 <---> first character
0000000: ff00 1700 4100 2000 4d00 6900 6e00 6500  ....A. .M.i.n.e.
0000010: 6300 7200 6100 6600 7400 2000 5300 6500  c.r.a.f.t. .S.e.
0000020: 7200 7600 6500 7200 a700 3000 a700 3100  r.v.e.r.§.0.§.1.
0000030: 30                                       0

[[Category:Minecraft Modern]]
