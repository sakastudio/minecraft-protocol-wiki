https://wiki.vg/Protocol
{{Box
  |BORDER = #9999FF
  |BACKGROUND = #99CCFF
  |WIDTH = 100%
  |ICON = 
  |HEADING = Heads up!
  |CONTENT = This article is about the protocol for the latest '''stable''' release of Minecraft '''Java Edition''' ([[Protocol version numbers|1.16.5, protocol 754]]). For the Java Edition pre-releases, see [[Pre-release protocol]]. For the incomplete Bedrock Edition docs, see [[Bedrock Protocol]]. For the old Pocket Edition, see [[Pocket Edition Protocol Documentation]].
}}

This page presents a dissection of the current '''[https://minecraft.net/ Minecraft] protocol'''.

If you're having trouble, check out the [[Protocol FAQ|FAQ]] or ask for help in the IRC channel [ircs://chat.freenode.net:6697/mcdevs #mcdevs on chat.freenode.net] ([http://wiki.vg/MCDevs More Information]).

'''Note:''' While you may use the contents of this page without restriction to create servers, clients, bots, etc… you still need to provide attribution to #mcdevs if you copy any of the contents of this page for publication elsewhere.

The changes between versions may be viewed at [[Protocol History]].

== Definitions ==

The Minecraft server accepts connections from TCP clients and communicates with them using ''packets''. A packet is a sequence of bytes sent over the TCP connection. The meaning of a packet depends both on its packet ID and the current state of the connection. The initial state of each connection is [[#Handshaking|Handshaking]], and state is switched using the packets [[#Handshake|Handshake]] and [[#Login Success|Login Success]].

=== Data types ===

{{:Data types}} <!-- Transcluded contents of Data types article in here — go to that page if you want to edit it -->

=== Other definitions ===

{| class="wikitable"
 |-
 ! Term
 ! Definition
 |-
 | Player
 | When used in the singular, Player always refers to the client connected to the server.
 |-
 | Entity
 | Entity refers to any item, player, mob, minecart or boat etc. See {{Minecraft Wiki|Entity|the Minecraft Wiki article}} for a full list.
 |-
 | EID
 | An EID — or Entity ID — is a 4-byte sequence used to identify a specific entity. An entity's EID is unique on the entire server.
 |-
 | XYZ
 | In this document, the axis names are the same as those shown in the debug screen (F3). Y points upwards, X points east, and Z points south.
 |-
 | Meter
 | The meter is Minecraft's base unit of length, equal to the length of a vertex of a solid block. The term “block” may be used to mean “meter” or “cubic meter”.
 |-
 | Global palette
 | A table/dictionary/palette mapping nonnegative integers to block states. Block state IDs are created in a linear fashion based off of order of assignment.  One block state ID allocated for each unique block state for a block; if a block has multiple properties then the number of allocated states is the product of the number of values for each property.  A current list of properties and state ID ranges is found on [https://pokechu22.github.io/Burger/1.16.5.html burger].

Alternatively, the vanilla server now includes an option to export the current block state ID mapping, by running <code>java -cp minecraft_server.jar net.minecraft.data.Main --reports</code>.  See [[Data Generators]] for more information.
 |-
 | Notchian
 | The official implementation of vanilla Minecraft as developed and released by Mojang.
 |}

== Packet format ==

Packets cannot be larger than 2097151 bytes (the maximum that can be sent in a 3-byte VarInt).  For compressed packets, this applies to both the compressed length and uncompressed lengths.

=== Without compression ===

{| class="wikitable"
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | Length
 | VarInt
 | Length of Packet ID + Data
 |-
 | Packet ID
 | VarInt
 | 
 |-
 | Data
 | Byte Array
 | Depends on the connection state and packet ID, see the sections below
 |}

=== With compression ===

Once a [[#Set Compression|Set Compression]] packet (with a non-negative threshold) is sent, [[wikipedia:Zlib|zlib]] compression is enabled for all following packets. The format of a packet changes slightly to include the size of the uncompressed packet.

{| class=wikitable
 ! Compressed?
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | No
 | Packet Length
 | VarInt
 | Length of Data Length + compressed length of (Packet ID + Data)
 |-
 | No
 | Data Length
 | VarInt
 | Length of uncompressed (Packet ID + Data) or 0
 |-
 |rowspan="2"| Yes
 | Packet ID
 | VarInt
 | zlib compressed packet ID (see the sections below)
 |-
 | Data
 | Byte Array
 | zlib compressed packet data (see the sections below)
 |}

The length given by the Packet Length field is the number of bytes that remain in that packet, including the Data Length field.

If Data Length is set to zero, then the packet is uncompressed; otherwise it is the size of the uncompressed packet.

If compressed, the uncompressed length of (Packet ID + Data) must be equal to or over the threshold set in the packet [[#Set Compression|Set Compression]], otherwise the receiving party will disconnect.

Compression can be disabled by sending the packet [[#Set Compression|Set Compression]] with a negative Threshold, or not sending the Set Compression packet at all.

== Handshaking ==

=== Clientbound ===

There are no clientbound packets in the Handshaking state, since the protocol immediately switches to a different state after the client sends the first packet.

=== Serverbound ===

==== Handshake ====

This causes the server to switch into the target state.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x00
 |rowspan="4"| Handshaking
 |rowspan="4"| Server
 | Protocol Version
 | VarInt
 | See [[protocol version numbers]] (currently 754 in Minecraft 1.16.5).
 |-
 | Server Address
 | String (255)
 | Hostname or IP, e.g. localhost or 127.0.0.1, that was used to connect. The Notchian server does not use this information. Note that SRV records are a complete redirect, e.g. if _minecraft._tcp.example.com points to mc.example.org, users connecting to example.com will provide mc.example.org as server address in addition to connecting to it.
 |- 
 | Server Port
 | Unsigned Short
 | Default is 25565. The Notchian server does not use this information.
 |-
 | Next State
 | VarInt Enum
 | 1 for [[#Status|status]], 2 for [[#Login|login]].
 |}

==== Legacy Server List Ping ====

{{Warning|This packet uses a nonstandard format. It is never length-prefixed, and the packet ID is an Unsigned Byte instead of a VarInt.}}

While not technically part of the current protocol, legacy clients may send this packet to initiate [[Server List Ping]], and modern servers should handle it correctly.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0xFE
 | Handshaking
 | Server
 | Payload
 | Unsigned Byte
 | always 1 (<code>0x01</code>).
 |}

See [[Server List Ping#1.6]] for the details of the protocol that follows this packet.

== Status ==
{{Main|Server List Ping}}

=== Clientbound ===

==== Response ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x00
 |rowspan="2"| Status
 |rowspan="2"| Client
 |-
 | JSON Response
 | String (32767)
 | See [[Server List Ping#Response]]; as with all strings this is prefixed by its length as a VarInt.
 |}

==== Pong ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x01
 | Status
 | Client
 | Payload
 | Long
 | Should be the same as sent by the client.
 |}

=== Serverbound ===

==== Request ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x00
 | Status
 | Server
 |colspan="3"| ''no fields''
 |}

==== Ping ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x01
 | Status
 | Server
 | Payload
 | Long
 | May be any number. Notchian clients use a system-dependent time value which is counted in milliseconds.
 |}

== Login ==

The login process is as follows:

# C→S: [[#Handshake|Handshake]] with Next State set to 2 (login)
# C→S: [[#Login Start|Login Start]]
# S→C: [[#Encryption Request|Encryption Request]]
# Client auth
# C→S: [[#Encryption Response|Encryption Response]]
# Server auth, both enable encryption
# S→C: [[#Set Compression|Set Compression]] (optional)
# S→C: [[#Login Success|Login Success]]

Set Compression, if present, must be sent before Login Success. Note that anything sent after Set Compression must use the [[#With_compression|Post Compression packet format]].

For unauthenticated ("cracked"/offline-mode) and localhost connections (either of the two conditions is enough for an unencrypted connection) there is no encryption. In that case [[#Login Start|Login Start]] is directly followed by [[#Login Success|Login Success]]. The Notchian server uses UUID v3 for offline player UUIDs, with the namespace “OfflinePlayer” and the value as the player’s username. For example, Notch’s offline UUID would be derived from the string “OfflinePlayer:Notch”. This is not a requirement however, the UUID may be anything.

See [[Protocol Encryption]] for details.

=== Clientbound ===

==== Disconnect (login) ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x00
 | Login
 | Client
 | Reason
 | [[Chat]]
 | 
 |}

==== Encryption Request ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x01
 |rowspan="5"| Login
 |rowspan="5"| Client
 | Server ID
 | String (20)
 | Appears to be empty.
 |-
 | Public Key Length
 | VarInt
 | Length of Public Key
 |-
 | Public Key
 | Byte Array
 | The server's public key in bytes
 |-
 | Verify Token Length
 | VarInt
 | Length of Verify Token. Always 4 for Notchian servers.
 |-
 | Verify Token
 | Byte Array
 | A sequence of random bytes generated by the server.
 |}

See [[Protocol Encryption]] for details.

==== Login Success ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x02
 |rowspan="2"| Login
 |rowspan="2"| Client
 | UUID
 | UUID
 | 
 |-
 | Username
 | String (16)
 | 
 |}

This packet switches the connection state to [[#Play|play]].

{{Warning2|The (notchian) server might take a bit to fully transition to the Play state, so it's recommended to wait before sending Play packets, either by setting a timeout, or waiting for Play packets from the server (usually [[#Player Info|Player Info]]).

The notchian client doesn't send any (non-[[#Keep Alive|keep alive]]) packets until the next tick/[[#Time Update|time update]] packet.
}}

==== Set Compression ====

Enables compression.  If compression is enabled, all following packets are encoded in the [[#With compression|compressed packet format]].  Negative or zero values will disable compression, meaning the packet format should remain in the [[#Without compression|uncompressed packet format]].  However, this packet is entirely optional, and if not sent, compression will also not be enabled (the notchian server does not send the packet when compression is disabled).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x03
 | Login
 | Client
 | Threshold
 | VarInt
 | Maximum size of a packet before it is compressed.
 |}

==== Login Plugin Request ====

Used to implement a custom handshaking flow together with [[#Login_Plugin_Response | Login Plugin Response]].

Unlike plugin messages in "play" mode, these messages follow a lock-step request/response scheme, where the client is expected to respond to a request indicating whether it understood. The notchian client always responds that it hasn't understood, and sends an empty payload.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x04
 |rowspan="3"| Login
 |rowspan="3"| Client
 | Message ID
 | VarInt
 | Generated by the server - should be unique to the connection.
 |-
 | Channel
 | Identifier
 | Name of the [[plugin channel]] used to send the data.
 |-
 | Data
 | Byte Array
 | Any data, depending on the channel. The length of this array must be inferred from the packet length.
 |}

=== Serverbound ===

==== Login Start  ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x00
 | Login
 | Server
 | Name
 | String (16)
 | Player's Username.
 |}

==== Encryption Response ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x01
 |rowspan="4"| Login
 |rowspan="4"| Server
 | Shared Secret Length
 | VarInt
 | Length of Shared Secret.
 |-
 | Shared Secret
 | Byte Array
 | Shared Secret value, encrypted with the server's public key.
 |-
 | Verify Token Length
 | VarInt
 | Length of Verify Token.
 |-
 | Verify Token
 | Byte Array
 | Verify Token value, encrypted with the same public key as the shared secret.
 |}

See [[Protocol Encryption]] for details.

==== Login Plugin Response ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x02
 |rowspan="3"| Login
 |rowspan="3"| Server
 | Message ID
 | VarInt
 | Should match ID from server.
 |-
 | Successful
 | Boolean
 | <code>true</code> if the client understands the request, <code>false</code> otherwise. When <code>false</code>, no payload follows.
 |-
 | Data
 | Optional Byte Array
 | Any data, depending on the channel. The length of this array must be inferred from the packet length.
 |}

== Play ==

=== Clientbound ===

==== Spawn Entity ====

Sent by the server when a vehicle or other non-living entity is created.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="12"| 0x00
 |rowspan="12"| Play
 |rowspan="12"| Client
 | Entity ID
 | VarInt
 | EID of the entity.
 |-
 | Object UUID
 | UUID
 | 
 |-
 | Type
 | VarInt
 | The type of entity (same as in [[#Spawn Living Entity|Spawn Living Entity]]).
 |-
 | X
 | Double
 | 
 |-
 | Y
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Pitch
 | Angle
 | 
 |-
 | Yaw
 | Angle
 | 
 |-
 | Data
 | Int
 | Meaning dependent on the value of the Type field, see [[Object Data]] for details.
 |-
 | Velocity X
 | Short
 |rowspan="3"| Same units as [[#Entity Velocity|Entity Velocity]].  Always sent, but only used when Data is greater than 0 (except for some entities which always ignore it; see [[Object Data]] for details).
 |-
 | Velocity Y
 | Short
 |-
 | Velocity Z
 | Short
 |}

==== Spawn Experience Orb ====

Spawns one or more experience orbs.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x01
 |rowspan="5"| Play
 |rowspan="5"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | X
 | Double
 | 
 |-
 | Y
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Count
 | Short
 | The amount of experience this orb will reward once collected.
 |}

==== Spawn Living Entity ====

Sent by the server when a living entity is spawned.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="12"| 0x02
 |rowspan="12"| Play
 |rowspan="12"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Entity UUID
 | UUID
 | 
 |-
 | Type
 | VarInt
 | The type of mob. See [[Entities#Mobs]].
 |-
 | X
 | Double
 | 
 |-
 | Y
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Yaw
 | Angle
 | 
 |-
 | Pitch
 | Angle
 | 
 |-
 | Head Pitch
 | Angle
 | 
 |-
 | Velocity X
 | Short
 | Same units as [[#Entity Velocity|Entity Velocity]].
 |-
 | Velocity Y
 | Short
 | Same units as [[#Entity Velocity|Entity Velocity]].
 |-
 | Velocity Z
 | Short
 | Same units as [[#Entity Velocity|Entity Velocity]].
 |}

==== Spawn Painting ====

This packet shows location, name, and type of painting.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x03
 |rowspan="5"| Play
 |rowspan="5"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Entity UUID
 | UUID
 | 
 |-
 | Motive
 | VarInt
 | Painting's ID, see below.
 |-
 | Location
 | Position
 | Center coordinates (see below).
 |-
 | Direction
 | Byte Enum
 | Direction the painting faces (North = 2, South = 0, West = 1, East = 3).
 |}

Calculating the center of an image: given a (width × height) grid of cells, with <code>(0, 0)</code> being the top left corner, the center is <code>(max(0, width / 2 - 1), height / 2)</code>. E.g. <code>(1, 0)</code> for a 2×1 painting, or <code>(1, 2)</code> for a 4×4 painting.

List of paintings by coordinates in <code>paintings_kristoffer_zetterstrand.png</code> (where x and y are in pixels from the top left and width and height are in pixels or 16ths of a block):

{| class="wikitable"
 ! Name
 ! ID
 ! x
 ! y
 ! width
 ! height
 |-
 | <code>minecraft:kebab</code>
 | 0
 | 0
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:aztec</code>
 | 1
 | 16
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:alban</code>
 | 2
 | 32
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:aztec2</code>
 | 3
 | 48
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:bomb</code>
 | 4
 | 64
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:plant</code>
 | 5
 | 80
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:wasteland</code>
 | 6
 | 96
 | 0
 | 16
 | 16
 |-
 | <code>minecraft:pool</code>
 | 7
 | 0
 | 32
 | 32
 | 16
 |-
 | <code>minecraft:courbet</code>
 | 8
 | 32
 | 32
 | 32
 | 16
 |-
 | <code>minecraft:sea</code>
 | 9
 | 64
 | 32
 | 32
 | 16
 |-
 | <code>minecraft:sunset</code>
 | 10
 | 96
 | 32
 | 32
 | 16
 |-
 | <code>minecraft:creebet</code>
 | 11
 | 128
 | 32
 | 32
 | 16
 |-
 | <code>minecraft:wanderer</code>
 | 12
 | 0
 | 64
 | 16
 | 32
 |-
 | <code>minecraft:graham</code>
 | 13
 | 16
 | 64
 | 16
 | 32
 |-
 | <code>minecraft:match</code>
 | 14
 | 0
 | 128
 | 32
 | 32
 |-
 | <code>minecraft:bust</code>
 | 15
 | 32
 | 128
 | 32
 | 32
 |-
 | <code>minecraft:stage</code>
 | 16
 | 64
 | 128
 | 32
 | 32
 |-
 | <code>minecraft:void</code>
 | 17
 | 96
 | 128
 | 32
 | 32
 |-
 | <code>skull_and_roses</code>
 | 18
 | 128
 | 128
 | 32
 | 32
 |-
 | <code>minecraft:wither</code>
 | 19
 | 160
 | 128
 | 32
 | 32
 |-
 | <code>minecraft:fighters</code>
 | 20
 | 0
 | 96
 | 64
 | 32
 |-
 | <code>minecraft:pointer</code>
 | 21
 | 0
 | 192
 | 64
 | 64
 |-
 | <code>minecraft:pigscene</code>
 | 22
 | 64
 | 192
 | 64
 | 64
 |-
 | <code>minecraft:burning_skull</code>
 | 23
 | 128
 | 192
 | 64
 | 64
 |-
 | <code>minecraft:skeleton</code>
 | 24
 | 192
 | 64
 | 64
 | 48
 |-
 | <code>minecraft:donkey_kong</code>
 | 25
 | 192
 | 112
 | 64
 | 48
 |}

The {{Minecraft Wiki|Painting#Canvases|Minecraft Wiki article on paintings}} also provides a list of painting names to the actual images.

==== Spawn Player ====

This packet is sent by the server when a player comes into visible range, ''not'' when a player joins.

This packet must be sent after the [[#Player Info|Player Info]] packet that adds the player data for the client to use when spawning a player. If the Player Info for the player spawned by this packet is not present when this packet arrives, Notchian clients will not spawn the player entity. The Player Info packet includes skin/cape data.

Servers can, however, safely spawn player entities for players not in visible range. The client appears to handle it correctly.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x04
 |rowspan="7"| Play
 |rowspan="7"| Client
 | Entity ID
 | VarInt
 | Player's EID.
 |-
 | Player UUID
 | UUID
 | See below for notes on {{Minecraft Wiki|Server.properties#online-mode|offline mode}} and NPCs.
 |-
 | X
 | Double
 | 
 |-
 | Y
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Yaw
 | Angle
 | 
 |-
 | Pitch
 | Angle
 | 
 |}

When in {{Minecraft Wiki|Server.properties#online-mode|online mode}}, the UUIDs must be valid and have valid skin blobs.

In offline mode, [[Wikipedia:Universally unique identifier#Versions 3 and 5 (namespace name-based)|UUID v3]] is used with the String <code>OfflinePlayer:&lt;player name&gt;</code>, encoded in UTF-8 (and case-sensitive).  The Notchain server uses <code>UUID.nameUUIDFromBytes</code>, implemented by OpenJDK [https://github.com/AdoptOpenJDK/openjdk-jdk8u/blob/9a91972c76ddda5c1ce28b50ca38cbd8a30b7a72/jdk/src/share/classes/java/util/UUID.java#L153-L175 here].

For NPCs UUID v2 should be used. Note:

 <+Grum> i will never confirm this as a feature you know that :)

In an example UUID, <code>xxxxxxxx-xxxx-Yxxx-xxxx-xxxxxxxxxxxx</code>, the UUID version is specified by <code>Y</code>. So, for UUID v3, <code>Y</code> will always be <code>3</code>, and for UUID v2, <code>Y</code> will always be <code>2</code>.

==== Entity Animation (clientbound) ====

Sent whenever an entity should change animation.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x05
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Entity ID
 | VarInt
 | Player ID.
 |-
 | Animation
 | Unsigned Byte
 | Animation ID (see below).
 |}

Animation can be one of the following values:

{| class="wikitable"
 ! ID
 ! Animation
 |-
 | 0
 | Swing main arm
 |-
 | 1
 | Take damage
 |-
 | 2
 | Leave bed
 |-
 | 3
 | Swing offhand
 |-
 | 4
 | Critical effect
 |-
 | 5
 | Magic critical effect
 |}

==== Statistics ====

Sent as a response to [[#Client_Status|Client Status 0x04]] (id 1). Will only send the changed values if previously requested.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To 
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="4"| 0x06
 |rowspan="4"| Play
 |rowspan="4"| Client
 |colspan="2"| Count
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="3"| Statistic
 | Category ID
 |rowspan="3"| Array
 | VarInt
 | See below.
 |-
 | Statistic ID
 | VarInt
 | See below.
 |-
 | Value
 | VarInt
 | The amount to set it to.
 |}

Categories (these are namespaced, but with <code>:</code> replaced with <code>.</code>):

{| class="wikitable"
 ! Name
 ! ID
 ! Registry
 |-
 | <code>minecraft.mined</code>
 | 0
 | Blocks
 |-
 | <code>minecraft.crafted</code>
 | 1
 | Items
 |-
 | <code>minecraft.used</code>
 | 2
 | Items
 |-
 | <code>minecraft.broken</code>
 | 3
 | Items
 |-
 | <code>minecraft.picked_up</code>
 | 4
 | Items
 |-
 | <code>minecraft.dropped</code>
 | 5
 | Items
 |-
 | <code>minecraft.killed</code>
 | 6
 | Entities
 |-
 | <code>minecraft.killed_by</code>
 | 7
 | Entities
 |-
 | <code>minecraft.custom</code>
 | 8
 | Custom
 |}

Blocks, Items, and Entities use block (not block state), item, and entity ids.

Custom has the following (unit only matters for clients):

{| class="wikitable"
 ! Name
 ! ID
 ! Unit
 |-
 | <code>minecraft.leave_game</code>
 | 0
 | None
 |-
 | <code>minecraft.play_one_minute</code>
 | 1
 | Time
 |-
 | <code>minecraft.time_since_death</code>
 | 2
 | Time
 |-
 | <code>minecraft.time_since_rest</code>
 | 3
 | Time
 |-
 | <code>minecraft.sneak_time</code>
 | 4
 | Time
 |-
 | <code>minecraft.walk_one_cm</code>
 | 5
 | Distance
 |-
 | <code>minecraft.crouch_one_cm</code>
 | 6
 | Distance
 |-
 | <code>minecraft.sprint_one_cm</code>
 | 7
 | Distance
 |-
 | <code>minecraft.walk_on_water_one_cm</code>
 | 8
 | Distance
 |-
 | <code>minecraft.fall_one_cm</code>
 | 9
 | Distance
 |-
 | <code>minecraft.climb_one_cm</code>
 | 10
 | Distance
 |-
 | <code>minecraft.fly_one_cm</code>
 | 11
 | Distance
 |-
 | <code>minecraft.walk_under_water_one_cm</code>
 | 12
 | Distance
 |-
 | <code>minecraft.minecart_one_cm</code>
 | 13
 | Distance
 |-
 | <code>minecraft.boat_one_cm</code>
 | 14
 | Distance
 |-
 | <code>minecraft.pig_one_cm</code>
 | 15
 | Distance
 |-
 | <code>minecraft.horse_one_cm</code>
 | 16
 | Distance
 |-
 | <code>minecraft.aviate_one_cm</code>
 | 17
 | Distance
 |-
 | <code>minecraft.swim_one_cm</code>
 | 18
 | Distance
 |-
 | <code>minecraft.strider_one_cm</code>
 | 19
 | Distance
 |-
 | <code>minecraft.jump</code>
 | 20
 | None
 |-
 | <code>minecraft.drop</code>
 | 21
 | None
 |-
 | <code>minecraft.damage_dealt</code>
 | 22
 | Damage
 |-
 | <code>minecraft.damage_dealt_absorbed</code>
 | 23
 | Damage
 |-
 | <code>minecraft.damage_dealt_resisted</code>
 | 24
 | Damage
 |-
 | <code>minecraft.damage_taken</code>
 | 25
 | Damage
 |-
 | <code>minecraft.damage_blocked_by_shield</code>
 | 26
 | Damage
 |-
 | <code>minecraft.damage_absorbed</code>
 | 27
 | Damage
 |-
 | <code>minecraft.damage_resisted</code>
 | 28
 | Damage
 |-
 | <code>minecraft.deaths</code>
 | 29
 | None
 |-
 | <code>minecraft.mob_kills</code>
 | 30
 | None
 |-
 | <code>minecraft.animals_bred</code>
 | 31
 | None
 |-
 | <code>minecraft.player_kills</code>
 | 32
 | None
 |-
 | <code>minecraft.fish_caught</code>
 | 33
 | None
 |-
 | <code>minecraft.talked_to_villager</code>
 | 34
 | None
 |-
 | <code>minecraft.traded_with_villager</code>
 | 35
 | None
 |-
 | <code>minecraft.eat_cake_slice</code>
 | 36
 | None
 |-
 | <code>minecraft.fill_cauldron</code>
 | 37
 | None
 |-
 | <code>minecraft.use_cauldron</code>
 | 38
 | None
 |-
 | <code>minecraft.clean_armor</code>
 | 39
 | None
 |-
 | <code>minecraft.clean_banner</code>
 | 40
 | None
 |-
 | <code>minecraft.clean_shulker_box</code>
 | 41
 | None
 |-
 | <code>minecraft.interact_with_brewingstand</code>
 | 42
 | None
 |-
 | <code>minecraft.interact_with_beacon</code>
 | 43
 | None
 |-
 | <code>minecraft.inspect_dropper</code>
 | 44
 | None
 |-
 | <code>minecraft.inspect_hopper</code>
 | 45
 | None
 |-
 | <code>minecraft.inspect_dispenser</code>
 | 46
 | None
 |-
 | <code>minecraft.play_noteblock</code>
 | 47
 | None
 |-
 | <code>minecraft.tune_noteblock</code>
 | 48
 | None
 |-
 | <code>minecraft.pot_flower</code>
 | 49
 | None
 |-
 | <code>minecraft.trigger_trapped_chest</code>
 | 50
 | None
 |-
 | <code>minecraft.open_enderchest</code>
 | 51
 | None
 |-
 | <code>minecraft.enchant_item</code>
 | 52
 | None
 |-
 | <code>minecraft.play_record</code>
 | 53
 | None
 |-
 | <code>minecraft.interact_with_furnace</code>
 | 54
 | None
 |-
 | <code>minecraft.interact_with_crafting_table</code>
 | 55
 | None
 |-
 | <code>minecraft.open_chest</code>
 | 56
 | None
 |-
 | <code>minecraft.sleep_in_bed</code>
 | 57
 | None
 |-
 | <code>minecraft.open_shulker_box</code>
 | 58
 | None
 |-
 | <code>minecraft.open_barrel</code>
 | 59
 | None
 |-
 | <code>minecraft.interact_with_blast_furnace</code>
 | 60
 | None
 |-
 | <code>minecraft.interact_with_smoker</code>
 | 61
 | None
 |-
 | <code>minecraft.interact_with_lectern</code>
 | 62
 | None
 |-
 | <code>minecraft.interact_with_campfire</code>
 | 63
 | None
 |-
 | <code>minecraft.interact_with_cartography_table</code>
 | 64
 | None
 |-
 | <code>minecraft.interact_with_loom</code>
 | 65
 | None
 |-
 | <code>minecraft.interact_with_stonecutter</code>
 | 66
 | None
 |-
 | <code>minecraft.bell_ring</code>
 | 67
 | None
 |-
 | <code>minecraft.raid_trigger</code>
 | 68
 | None
 |-
 | <code>minecraft.raid_win</code>
 | 69
 | None
 |-
 | <code>minecraft.interact_with_anvil</code>
 | 70
 | None
 |-
 | <code>minecraft.interact_with_grindstone</code>
 | 71
 | None
 |-
 | <code>minecraft.target_hit</code>
 | 72
 | None
 |-
 | <code>minecraft.interact_with_smithing_table</code>
 | 73
 | None
 |}

Units:

* None: just a normal number (formatted with 0 decimal places)
* Damage: value is 10 times the normal amount
* Distance: a distance in centimeters (hundredths of blocks)
* Time: a time span in ticks

==== Acknowledge Player Digging ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x07
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Location
 | Position
 | Position where the digging was happening.
 |-
 | Block
 | VarInt
 | Block state ID of the block that should be at that position now.
 |-
 | Status
 | VarInt enum
 | Same as Player Digging.  Only Started digging (0), Cancelled digging (1), and Finished digging (2) are used.
 |-
 | Successful
 | Boolean
 | True if the digging succeeded; false if the client should undo any changes it made locally.
 |}

==== Block Break Animation ====

0–9 are the displayable destroy stages and each other number means that there is no animation on this coordinate.

Block break animations can still be applied on air; the animation will remain visible although there is no block being broken.  However, if this is applied to a transparent block, odd graphical effects may happen, including water losing its transparency.  (An effect similar to this can be seen in normal gameplay when breaking ice blocks)

If you need to display several break animations at the same time you have to give each of them a unique Entity ID. The entity ID does not need to correspond to an actual entity on the client. It is valid to use a randomly generated number.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x08
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Entity ID
 | VarInt
 | Entity ID of the entity breaking the block.
 |-
 | Location
 | Position
 | Block Position.
 |-
 | Destroy Stage
 | Byte
 | 0–9 to set it, any other value to remove it.
 |}

==== Block Entity Data ====

Sets the block entity associated with the block at the given location.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x09
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Location
 | Position
 | 
 |-
 | Action
 | Unsigned Byte
 | The type of update to perform, see below.
 |-
 | NBT Data
 | [[NBT|NBT Tag]]
 | Data to set.  May be a TAG_END (0), in which case the block entity at the given location is removed (though this is not required since the client will remove the block entity automatically on chunk unload or block removal).
 |}

''Action'' field:

* '''1''': Set data of a mob spawner (everything except for SpawnPotentials: current delay, min/max delay, mob to be spawned, spawn count, spawn range, etc.)
* '''2''': Set command block text (command and last execution status)
* '''3''': Set the level, primary, and secondary powers of a beacon
* '''4''': Set rotation and skin of mob head
* '''5''': Declare a conduit
* '''6''': Set base color and patterns on a banner
* '''7''': Set the data for a Structure tile entity
* '''8''': Set the destination for a end gateway
* '''9''': Set the text on a sign
* '''10''': Unused
* '''11''': Declare a bed
* '''12''': Set data of a jigsaw block
* '''13''': Set items in a campfire
* '''14''': Beehive information

==== Block Action ====

This packet is used for a number of actions and animations performed by blocks, usually non-persistent.

See [[Block Actions]] for a list of values.

{{Warning2|This packet uses a block ID, not a block state.}}

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x0A
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Location
 | Position
 | Block coordinates.
 |-
 | Action ID (Byte 1)
 | Unsigned Byte
 | Varies depending on block — see [[Block Actions]].
 |-
 | Action Param (Byte 2)
 | Unsigned Byte
 | Varies depending on block — see [[Block Actions]].
 |-
 | Block Type
 | VarInt
 | The block type ID for the block.  This must match the block at the given coordinates.
 |}

==== Block Change ====

Fired whenever a block is changed within the render distance.

{{Warning2|Changing a block in a chunk that is not loaded is not a stable action.  The Notchian client currently uses a ''shared'' empty chunk which is modified for all block changes in unloaded chunks; while in 1.9 this chunk never renders in older versions the changed block will appear in all copies of the empty chunk.  Servers should avoid sending block changes in unloaded chunks and clients should ignore such packets.}}

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x0B
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Location
 | Position
 | Block Coordinates.
 |-
 | Block ID
 | VarInt
 | The new block state ID for the block as given in the {{Minecraft Wiki|Data values#Block IDs|global palette}}. See that section for more information.
 |}

==== Boss Bar ==== 

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="14"| 0x0C
 |rowspan="14"| Play
 |rowspan="14"| Client
 |colspan="2"| UUID
 | UUID
 | Unique ID for this bar.
 |-
 |colspan="2"| Action
 | VarInt Enum
 | Determines the layout of the remaining packet.
 |-
 ! Action
 ! Field Name
 ! 
 ! 
 |-
 |rowspan="5"| 0: add
 | Title
 | [[Chat]]
 | 
 |-
 | Health
 | Float
 | From 0 to 1. Values greater than 1 do not crash a Notchian client, and start [https://i.johni0702.de/nA.png rendering part of a second health bar] at around 1.5.
 |-
 | Color
 | VarInt Enum
 | Color ID (see below).
 |-
 | Division
 | VarInt Enum
 | Type of division (see below).
 |-
 | Flags
 | Unsigned Byte
 | Bit mask. 0x1: should darken sky, 0x2: is dragon bar (used to play end music), 0x04: create fog (previously was also controlled by 0x02).
 |-
 | 1: remove
 | ''no fields''
 | ''no fields''
 | Removes this boss bar.
 |-
 | 2: update health
 | Health
 | Float
 | ''as above''
 |-
 | 3: update title
 | Title
 | [[Chat]]
 | 
 |-
 |rowspan="2"| 4: update style
 | Color
 | VarInt Enum
 | Color ID (see below).
 |-
 | Dividers
 | VarInt Enum
 | ''as above''
 |-
 | 5: update flags
 | Flags
 | Unsigned Byte
 | ''as above''
 |-
 |}

{| class="wikitable"
 ! ID
 ! Color
 |-
 | 0
 | Pink
 |-
 | 1
 | Blue
 |-
 | 2
 | Red
 |-
 | 3
 | Green
 |-
 | 4
 | Yellow
 |-
 | 5
 | Purple
 |-
 | 6
 | White
 |}

{| class="wikitable"
 ! ID
 ! Type of division
 |-
 | 0
 | No division
 |-
 | 1
 | 6 notches
 |-
 | 2
 | 10 notches
 |-
 | 3
 | 12 notches
 |-
 | 4
 | 20 notches
 |}

==== Server Difficulty ====

Changes the difficulty setting in the client's option menu

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x0D
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Difficulty
 | Unsigned Byte
 | 0: peaceful, 1: easy, 2: normal, 3: hard.
 |-
 | Difficulty locked?
 | Boolean
 |
 |}

==== Chat Message (clientbound) ==== 

Identifying the difference between Chat/System Message is important as it helps respect the user's chat visibility options.  See [[Chat#Processing chat|processing chat]] for more info about these positions.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x0E
 |rowspan="3"| Play
 |rowspan="3"| Client
 | JSON Data
 | [[Chat]]
 | Limited to 262144 bytes.
 |-
 | Position
 | Byte
 | 0: chat (chat box), 1: system message (chat box), 2: game info (above hotbar).
 |-
 | Sender
 | UUID
 | Used by the Notchian client for the disableChat launch option. Setting both longs to 0 will always display the message regardless of the setting.
 |}

==== Tab-Complete (clientbound) ====

The server responds with a list of auto-completions of the last word sent to it. In the case of regular chat, this is a player username. Command names and parameters are also supported. The client sorts these alphabetically before listing them.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="8"| 0x0F
 |rowspan="8"| Play
 |rowspan="8"| Client
 |-
 |colspan="2"| ID
 |colspan="2"| VarInt
 | Transaction ID.
 |-
 |colspan="2"| Start
 |colspan="2"| VarInt
 | Start of the text to replace.
 |-
 |colspan="2"| Length
 |colspan="2"| VarInt
 | Length of the text to replace.
 |-
 |colspan="2"| Count
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="3"| Matches
 | Match
 |rowspan="3"| Array
 | String (32767)
 | One eligible value to insert, note that each command is sent separately instead of in a single string, hence the need for Count.  Note that for instance this doesn't include a leading <code>/</code> on commands.
 |-
 | Has tooltip
 | Boolean
 | True if the following is present.
 |-
 | Tooltip
 | Optional [[Chat]]
 | Tooltip to display; only present if previous boolean is true.
 |}

==== Declare Commands ====

Lists all of the commands on the server, and how they are parsed.

This is a directed graph, with one root node.  Each redirect or child node must refer only to nodes that have already been declared.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x10
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Nodes
 | Array of [[Command Data|Node]]
 | An array of nodes.
 |-
 | Root index
 | VarInt
 | Index of the <code>root</code> node in the previous array.
 |}

For more information on this packet, see the [[Command Data]] article.

==== Window Confirmation (clientbound) ====

A packet from the server indicating whether a request from the client was accepted, or whether there was a conflict (due to lag).  If the packet was not accepted, the client must respond with a [[#Window Confirmation (serverbound)|serverbound window confirmation]] packet.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x11
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID
 | Byte
 | The ID of the window that the action occurred in.
 |-
 | Action Number
 | Short
 | Every action that is to be accepted has a unique number. This number is an incrementing integer (starting at 0) with separate counts for each window ID.
 |-
 | Accepted
 | Boolean
 | Whether the action was accepted.
 |}

==== Close Window (clientbound) ====

This packet is sent from the server to the client when a window is forcibly closed, such as when a chest is destroyed while it's open. The notchian client disregards the provided window ID and closes any active window.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x12
 | Play
 | Client
 | Window ID
 | Unsigned Byte
 | This is the ID of the window that was closed. 0 for inventory.
 |}

==== Window Items ====
[[File:Inventory-slots.png|thumb|The inventory slots]]

Sent by the server when items in multiple slots (in a window) are added/removed. This includes the main inventory, equipped armour and crafting slots. This packet with Window ID set to "0" is sent during the player joining sequence to initialise the player's inventory. 

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x13
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID
 | Unsigned Byte
 | The ID of window which items are being sent for. 0 for player inventory.
 |-
 | Count
 | Short
 | Number of elements in the following array.
 |-
 | Slot Data
 | Array of [[Slot Data|Slot]]
 | 
 |}

See [[Inventory#Windows|inventory windows]] for further information about how slots are indexed.

==== Window Property ====

This packet is used to inform the client that part of a GUI window should be updated.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x14
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID
 | Unsigned Byte
 | 
 |-
 | Property
 | Short
 | The property to be updated, see below.
 |-
 | Value
 | Short
 | The new value for the property, see below.
 |}

The meaning of the Property field depends on the type of the window. The following table shows the known combinations of window type and property, and how the value is to be interpreted.

{| class="wikitable"
 |-
 ! Window type
 ! Property
 ! Value
 |-
 |rowspan="4"| Furnace
 | 0: Fire icon (fuel left)
 | counting from fuel burn time down to 0 (in-game ticks)
 |-
 | 1: Maximum fuel burn time
 | fuel burn time or 0 (in-game ticks)
 |-
 | 2: Progress arrow
 | counting from 0 to maximum progress (in-game ticks)
 |-
 | 3: Maximum progress
 | always 200 on the notchian server
 |-
 |rowspan="10"| Enchantment Table
 | 0: Level requirement for top enchantment slot
 |rowspan="3"| The enchantment's xp level requirement
 |-
 | 1: Level requirement for middle enchantment slot
 |-
 | 2: Level requirement for bottom enchantment slot
 |-
 | 3: The enchantment seed
 | Used for drawing the enchantment names (in [[Wikipedia:Standard Galactic Alphabet|SGA]]) clientside.  The same seed ''is'' used to calculate enchantments, but some of the data isn't sent to the client to prevent easily guessing the entire list (the seed value here is the regular seed bitwise and <code>0xFFFFFFF0</code>).
 |-
 | 4: Enchantment ID shown on mouse hover over top enchantment slot
 |rowspan="3"| The enchantment id (set to -1 to hide it), see below for values
 |-
 | 5: Enchantment ID shown on mouse hover over middle enchantment slot
 |-
 | 6: Enchantment ID shown on mouse hover over bottom enchantment slot
 |-
 | 7: Enchantment level shown on mouse hover over the top slot
 |rowspan="3"| The enchantment level (1 = I, 2 = II, 6 = VI, etc.), or -1 if no enchant
 |-
 | 8: Enchantment level shown on mouse hover over the middle slot
 |-
 | 9: Enchantment level shown on mouse hover over the bottom slot
 |-
 |rowspan="3"| Beacon
 | 0: Power level
 | 0-4, controls what effect buttons are enabled
 |-
 | 1: First potion effect
 | {{Minecraft Wiki|Data values#Status effects|Potion effect ID}} for the first effect, or -1 if no effect
 |-
 | 2: Second potion effect
 | {{Minecraft Wiki|Data values#Status effects|Potion effect ID}} for the second effect, or -1 if no effect
 |-
 | Anvil
 | 0: Repair cost
 | The repair's cost in xp levels
 |-
 |rowspan="2"| Brewing Stand
 | 0: Brew time
 | 0 – 400, with 400 making the arrow empty, and 0 making the arrow full
 |-
 | 1: Fuel time
 | 0 - 20, with 0 making the arrow empty, and 20 making the arrow full
 |-
 | Stonecutter
 | 0: Selected recipe
 | The index of the selected recipe. -1 means none is selected.
 |-
 | Loom
 | 0: Selected pattern
 | The index of the selected pattern. 0 means none is selected, 0 is also the internal id of the "base" pattern.
 |-
 | Lectern
 | 0: Page number
 | The current page number, starting from 0.
 |}

For an enchanting table, the following numerical IDs are used:

{| class="wikitable"
 ! Numerical ID
 ! Enchantment ID
 ! Enchantment Name
 |-
 | 0
 | minecraft:protection
 | Protection
 |-
 | 1
 | minecraft:fire_protection
 | Fire Protection
 |-
 | 2
 | minecraft:feather_falling
 | Feather Falling
 |-
 | 3
 | minecraft:blast_protection
 | Blast Protection
 |-
 | 4
 | minecraft:projectile_protection
 | Projectile Protection
 |-
 | 5
 | minecraft:respiration
 | Respiration
 |-
 | 6
 | minecraft:aqua_affinity
 | Aqua Affinity
 |-
 | 7
 | minecraft:thorns
 | Thorns
 |-
 | 8
 | minecraft:depth_strider
 | Depth Strider
 |-
 | 9
 | minecraft:frost_walker
 | Frost Walker
 |-
 | 10
 | minecraft:binding_curse
 | Curse of Binding
 |-
 | 11
 | minecraft:soul_speed
 | Soul Speed
 |-
 | 12
 | minecraft:sharpness
 | Sharpness
 |-
 | 13
 | minecraft:smite
 | Smite
 |-
 | 14
 | minecraft:bane_of_arthropods
 | Bane of Arthropods
 |-
 | 15
 | minecraft:knockback
 | Knockback
 |-
 | 16
 | minecraft:fire_aspect
 | Fire Aspect
 |-
 | 17
 | minecraft:looting
 | Looting
 |-
 | 18
 | minecraft:sweeping
 | Sweeping Edge
 |-
 | 19
 | minecraft:efficiency
 | Efficiency
 |-
 | 20
 | minecraft:silk_touch
 | Silk Touch
 |-
 | 21
 | minecraft:unbreaking
 | Unbreaking
 |-
 | 22
 | minecraft:fortune
 | Fortune
 |-
 | 23
 | minecraft:power
 | Power
 |-
 | 24
 | minecraft:punch
 | Punch
 |-
 | 25
 | minecraft:flame
 | Flame
 |-
 | 26
 | minecraft:infinity
 | Infinity
 |-
 | 27
 | minecraft:luck_of_the_sea
 | Luck of the Sea
 |-
 | 28
 | minecraft:lure
 | Lure
 |-
 | 29
 | minecraft:loyalty
 | Loyalty
 |-
 | 30
 | minecraft:impaling
 | Impaling
 |-
 | 31
 | minecraft:riptide
 | Riptide
 |-
 | 32
 | minecraft:channeling
 | Channeling
 |-
 | 33
 | minecraft:multishot
 | Multishot
 |-
 | 34
 | minecraft:quick_charge
 | Quick Charge
 |-
 | 35
 | minecraft:piercing
 | Piercing
 |-
 | 36
 | minecraft:mending
 | Mending
 |-
 | 37
 | minecraft:vanishing_curse
 | Curse of Vanishing
 |}

==== Set Slot ====

Sent by the server when an item in a slot (in a window) is added/removed.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x15
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID
 | Byte
 | The window which is being updated. 0 for player inventory. Note that all known window types include the player inventory. This packet will only be sent for the currently opened window while the player is performing actions, even if it affects the player inventory. After the window is closed, a number of these packets are sent to update the player's inventory window (0).
 |-
 | Slot
 | Short
 | The slot that should be updated.
 |-
 | Slot Data
 | [[Slot Data|Slot]]
 | 
 |}

To set the cursor (the item currently dragged with the mouse), use -1 as Window ID and as Slot.

This packet can only be used to edit the hotbar of the player's inventory if window ID is set to 0 (slots 36 through 44).  If the window ID is set to -2, then any slot in the inventory can be used but no add item animation will be played.

==== Set Cooldown ====

Applies a cooldown period to all items with the given type.  Used by the Notchian server with enderpearls.  This packet should be sent when the cooldown starts and also when the cooldown ends (to compensate for lag), although the client will end the cooldown automatically. Can be applied to any item, note that interactions still get sent to the server with the item but the client does not play the animation nor attempt to predict results (i.e block placing).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x16
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Item ID
 | VarInt
 | Numeric ID of the item to apply a cooldown to.
 |-
 | Cooldown Ticks
 | VarInt
 | Number of ticks to apply a cooldown for, or 0 to clear the cooldown.
 |}

==== Plugin Message (clientbound) ====

{{Main|Plugin channels}}

Mods and plugins can use this to send their data. Minecraft itself uses several [[plugin channel]]s. These internal channels are in the <code>minecraft</code> namespace.

More documentation on this: [http://dinnerbone.com/blog/2012/01/13/minecraft-plugin-channels-messaging/ http://dinnerbone.com/blog/2012/01/13/minecraft-plugin-channels-messaging/]

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x17
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Channel
 | Identifier
 | Name of the [[plugin channel]] used to send the data.
 |-
 | Data
 | Byte Array
 | Any data, depending on the channel. <code>minecraft:</code> channels are documented [[plugin channel|here]].  The length of this array must be inferred from the packet length.
 |}

==== Named Sound Effect ====
{{See also|#Sound Effect}}

Used to play a sound effect on the client. Custom sounds may be added by resource packs.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x18
 |rowspan="7"| Play
 |rowspan="7"| Client
 | Sound Name
 | Identifier
 | All sound effect names as of 1.16.5 can be seen [https://pokechu22.github.io/Burger/1.16.5.html#sounds here].
 |-
 | Sound Category
 | VarInt Enum
 | The category that this sound will be played from ([https://gist.github.com/konwboj/7c0c380d3923443e9d55 current categories]).
 |-
 | Effect Position X
 | Int
 | Effect X multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Effect Position Y
 | Int
 | Effect Y multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Effect Position Z
 | Int
 | Effect Z multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Volume
 | Float
 | 1 is 100%, can be more.
 |-
 | Pitch
 | Float
 | Float between 0.5 and 2.0 by Notchian clients.
 |}

==== Disconnect (play) ====

Sent by the server before it disconnects a client. The client assumes that the server has already closed the connection by the time the packet arrives.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x19
 | Play
 | Client
 | Reason
 | [[Chat]]
 | Displayed to the client when the connection terminates.
 |}

==== Entity Status ====

Entity statuses generally trigger an animation for an entity.  The available statuses vary by the entity's type (and are available to subclasses of that type as well).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x1A
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Entity ID
 | Int
 | 
 |-
 | Entity Status
 | Byte Enum
 | See [[Entity statuses]] for a list of which statuses are valid for each type of entity.
 |}

==== Explosion ====

Sent when an explosion occurs (creepers, TNT, and ghast fireballs).

Each block in Records is set to air. Coordinates for each axis in record is int(X) + record.x

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="9"| 0x1B
 |rowspan="9"| Play
 |rowspan="9"| Client
 | X
 | Float
 | 
 |-
 | Y
 | Float
 | 
 |-
 | Z
 | Float
 | 
 |-
 | Strength
 | Float
 | A strength greater than or equal to 2.0 spawns a <code>minecraft:explosion_emitter</code> particle, while a lesser strength spawns a <code>minecraft:explosion</code> particle.
 |-
 | Record Count
 | Int
 | Number of elements in the following array.
 |-
 | Records
 | Array of (Byte, Byte, Byte)
 | Each record is 3 signed bytes long; the 3 bytes are the XYZ (respectively) signed offsets of affected blocks.
 |-
 | Player Motion X
 | Float
 | X velocity of the player being pushed by the explosion.
 |-
 | Player Motion Y
 | Float
 | Y velocity of the player being pushed by the explosion.
 |-
 | Player Motion Z
 | Float
 | Z velocity of the player being pushed by the explosion.
 |}

==== Unload Chunk ====

Tells the client to unload a chunk column.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x1C
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Chunk X
 | Int
 | Block coordinate divided by 16, rounded down.
 |-
 | Chunk Z
 | Int
 | Block coordinate divided by 16, rounded down.
 |}

It is legal to send this packet even if the given chunk is not currently loaded.

==== Change Game State ====

Used for a wide variety of game state things, from weather to bed use to gamemode to demo messages.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x1D
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Reason
 | Unsigned Byte
 | See below.
 |-
 | Value
 | Float
 | Depends on Reason.
 |}

''Reason codes'':

{| class="wikitable"
 ! Reason
 ! Effect
 ! Value
 |-
 | 0
 | No respawn block available
 | Note: Sends message 'block.minecraft.spawn.not_valid'(You have no home bed or charged respawn anchor, or it was obstructed) to the client.
 |-
 | 1
 | End raining
 | 
 |-
 | 2
 | Begin raining
 | 
 |-
 | 3
 | Change gamemode
 | 0: Survival, 1: Creative, 2: Adventure, 3: Spectator.
 |-
 | 4
 | Win game
 | 0: Just respawn player. <br>1: Roll the credits and respawn player. <br> Note that 1 is only sent by notchian server when player has not yet achieved advancement "The end?", else 0 is sent.
 |-
 | 5
 | Demo event
 | 0: Show welcome to demo screen<br> 101: Tell movement controls<br> 102: Tell jump control<br> 103: Tell inventory control<br> 104: Tell that the demo is over and print a message about how to take a screenshot.
 |- 
 | 6
 | Arrow hit player
 | Note: Sent when any player is struck by an arrow. 
 |-
 | 7
 | Rain level change
 | Note: Seems to change both skycolor and lightning. <strong>[https://imgur.com/a/K6wSrkW You can see some level images here]</strong><br>This can cause [https://imgur.com/gallery/ZQX0Wd5 <strong>HUD color change in client</strong>], when level is higher than 20. It goes away only when game is restarted or client receives same packet (from any server) but with value of 0. Is this a bug?<br><br>Rain level starting from 0.
 |-
 | 8
 | Thunder level change
 | Note: Seems to change both skycolor and lightning (same as Rain level change, but doesn't start rain). It also requires rain to render by notchian client. <br>This can cause [https://imgur.com/gallery/ZQX0Wd5 <strong>HUD color change in client</strong>], when level is higher than 20. It goes away only when game is restarted or client receives same packet (from any server) but with value of 0. Is this a bug?<br><br>Thunder level starting from 0.
 |-
 | 9
 | Play pufferfish sting sound.
 |-
 | 10
 | Play elder guardian mob appearance (effect and sound).
 | 
 |-
 | 11
 | Enable respawn screen
 |  0: Enable respawn screen, 1: Immediately respawn (sent when the doImmediateRespawn gamerule changes).
 |}

==== Open Horse Window ====

This packet is used exclusively for opening the horse GUI.  [[#Open Window|Open Window]] is used for all other GUIs.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x1E
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID?
 | Byte
 |
 |-
 | Number of slots?
 | VarInt
 |
 |-
 | Entity ID?
 | Integer
 |
 |}

==== Keep Alive (clientbound) ====

The server will frequently send out a keep-alive, each containing a random ID. The client must respond with the same payload (see [[#Keep Alive (serverbound)|serverbound Keep Alive]]). If the client does not respond to them for over 30 seconds, the server kicks the client. Vice versa, if the server does not send any keep-alives for 20 seconds, the client will disconnect and yields a "Timed out" exception.

The Notchian server uses a system-dependent time in milliseconds to generate the keep alive ID value.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x1F
 | Play
 | Client
 | Keep Alive ID
 | Long
 | 
 |}

==== Chunk Data ====
{{Main|Chunk Format}}
{{See also|#Unload Chunk}}

{{Need Info|How do biomes work now?  The biome change happened at the same time as the seed change, but it's not clear how/if biomes could be computed given that it's not the actual seed...  ([https://www.reddit.com/r/Mojira/comments/e5at6i/a_discussion_for_the_changes_to_how_biomes_are/ /r/mojira discussion] which notes that it seems to be some kind of interpolation, and 3D biomes are only used in the nether)}}

The server only sends skylight information for chunk pillars in the {{Minecraft Wiki|Overworld}}, it's up to the client to know in which dimension the player is currently located. You can also infer this information from the primary bitmask and the amount of uncompressed bytes sent.  This packet also sends all block entities in the chunk (though sending them is not required; it is still legal to send them with [[#Block Entity Data|Block Entity Data]] later).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="11"| 0x20
 |rowspan="11"| Play
 |rowspan="11"| Client
 | Chunk X
 | Int
 | Chunk coordinate (block coordinate divided by 16, rounded down).
 |-
 | Chunk Z
 | Int
 | Chunk coordinate (block coordinate divided by 16, rounded down).
 |-
 | Full chunk
 | Boolean
 | See [[Chunk Format#Full chunk|Chunk Format]].
 |-
 | Primary Bit Mask
 | VarInt
 | Bitmask with bits set to 1 for every 16×16×16 chunk section whose data is included in Data. The least significant bit represents the chunk section at the bottom of the chunk column (from y=0 to y=15).
 |-
 | Heightmaps
 | [[NBT]]
 | Compound containing one long array named <code>MOTION_BLOCKING</code>, which is a heightmap for the highest solid block at each position in the chunk (as a compacted long array with 256 entries at 9 bits per entry totaling 36 longs). The Notchian server also adds a <code>WORLD_SURFACE</code> long array, the purpose of which is unknown, but it's not required for the chunk to be accepted.
 |- 
 | Biomes length
 | Optional VarInt
 | Size of the following array; should always be 1024.  Not present if full chunk is false.
 |-
 | Biomes
 | Optional array of VarInt
 | 1024 biome IDs, ordered by x then z then y, in 4&times;4&times;4 blocks.  Not present if full chunk is false.  See [[Chunk Format#Biomes|Chunk Format § Biomes]].
 |- 
 | Size
 | VarInt
 | Size of Data in bytes.
 |-
 | Data
 | Byte array
 | See [[Chunk Format#Data structure|data structure]] in Chunk Format.
 |-
 | Number of block entities
 | VarInt
 | Number of elements in the following array.
 |-
 | Block entities
 | Array of [[NBT|NBT Tag]]
 | All block entities in the chunk. Use the x, y, and z tags in the NBT to determine their positions.
 |}

Note that the Notchian client requires an [[#Update View Position|Update View Position]] packet when it crosses a chunk border, otherwise it'll only display render distance + 2 chunks around the chunk it spawned in.

The compacted array format has been adjusted so that individual entries no longer span across multiple longs, affecting the main data array and heightmaps.

New format, 5 bits per block, containing the following references to blocks in a palette (not shown): <span style="border: solid 2px hsl(0, 90%, 60%)">1</span><span style="border: solid 2px hsl(30, 90%, 60%)">2</span><span style="border: solid 2px hsl(60, 90%, 60%)">2</span><span style="border: solid 2px hsl(90, 90%, 60%)">3</span><span style="border: solid 2px hsl(120, 90%, 60%)">4</span><span style="border: solid 2px hsl(150, 90%, 60%)">4</span><span style="border: solid 2px hsl(180, 90%, 60%)">5</span><span style="border: solid 2px hsl(210, 90%, 60%)">6</span><span style="border: solid 2px hsl(240, 90%, 60%)">6</span><span style="border: solid 2px hsl(270, 90%, 60%)">4</span><span style="border: solid 2px hsl(300, 90%, 60%)">8</span><span style="border: solid 2px hsl(330, 90%, 60%)">0</span><span style="border: solid 2px hsl(0, 90%, 30%)">7</span><span style="border: solid 2px hsl(30, 90%, 30%)">4</span><span style="border: solid 2px hsl(60, 90%, 30%)">3</span><span style="border: solid 2px hsl(90, 90%, 30%)">13</span><span style="border: solid 2px hsl(120, 90%, 30%)">15</span><span style="border: solid 2px hsl(150, 90%, 30%)">16</span><span style="border: solid 2px hsl(180, 90%, 30%)">9</span><span style="border: solid 2px hsl(210, 90%, 30%)">14</span><span style="border: solid 2px hsl(240, 90%, 30%)">10</span><span style="border: solid 2px hsl(270, 90%, 30%)">12</span><span style="border: solid 2px hsl(300, 90%, 30%)">0</span><span style="border: solid 2px hsl(330, 90%, 30%)">2</span>

<code>0020863148418841</code> <code><span style="outline: dashed 2px black">0000</span><span style="outline: solid 2px hsl(330, 90%, 60%)">00000</span><span style="outline: solid 2px hsl(300, 90%, 60%)">01000</span><span style="outline: solid 2px hsl(270, 90%, 60%)">00100</span><span style="outline: solid 2px hsl(240, 90%, 60%)">00110</span><span style="outline: solid 2px hsl(210, 90%, 60%)">00110</span><span style="outline: solid 2px hsl(180, 90%, 60%)">00101</span><span style="outline: solid 2px hsl(150, 90%, 60%)">00100</span><span style="outline: solid 2px hsl(120, 90%, 60%)">00100</span><span style="outline: solid 2px hsl(90, 90%, 60%)">00011</span><span style="outline: solid 2px hsl(60, 90%, 60%)">00010</span><span style="outline: solid 2px hsl(30, 90%, 60%)">00010</span><span style="outline: solid 2px hsl(0, 90%, 60%)">00001</span></code><br>
<code>01018A7260F68C87</code> <code><span style="outline: dashed 2px black">0000</span><span style="outline: solid 2px hsl(330, 90%, 30%)">00010</span><span style="outline: solid 2px hsl(300, 90%, 30%)">00000</span><span style="outline: solid 2px hsl(270, 90%, 30%)">01100</span><span style="outline: solid 2px hsl(240, 90%, 30%)">01010</span><span style="outline: solid 2px hsl(210, 90%, 30%)">01110</span><span style="outline: solid 2px hsl(180, 90%, 30%)">01001</span><span style="outline: solid 2px hsl(150, 90%, 30%)">10000</span><span style="outline: solid 2px hsl(120, 90%, 30%)">01111</span><span style="outline: solid 2px hsl(90, 90%, 30%)">01101</span><span style="outline: solid 2px hsl(60, 90%, 30%)">00011</span><span style="outline: solid 2px hsl(30, 90%, 30%)">00100</span><span style="outline: solid 2px hsl(0, 90%, 30%)">00111</span></code>

==== Effect ====
Sent when a client is to play a sound or particle effect.

By default, the Minecraft client adjusts the volume of sound effects based on distance. The final boolean field is used to disable this, and instead the effect is played from 2 blocks away in the correct direction. Currently this is only used for effect 1023 (wither spawn), effect 1028 (enderdragon death), and effect 1038 (end portal opening); it is ignored on other effects.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x21
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Effect ID
 | Int
 | The ID of the effect, see below.
 |-
 | Location
 | Position
 | The location of the effect.
 |-
 | Data
 | Int
 | Extra data for certain effects, see below.
 |-
 | Disable Relative Volume
 | Boolean
 | See above.
 |}

Effect IDs:

{| class="wikitable"
 ! ID
 ! Name
 ! Data
 |-
 !colspan="3"| Sound
 |-
 | 1000
 | Dispenser dispenses
 | 
 |-
 | 1001
 | Dispenser fails to dispense
 | 
 |-
 | 1002
 | Dispenser shoots
 | 
 |-
 | 1003
 | Ender eye launched
 | 
 |-
 | 1004
 | Firework shot
 | 
 |-
 | 1005
 | Iron door opened
 | 
 |-
 | 1006
 | Wooden door opened
 | 
 |-
 | 1007
 | Wooden trapdoor opened
 | 
 |-
 | 1008
 | Fence gate opened
 | 
 |-
 | 1009
 | Fire extinguished
 | 
 |-
 | 1010
 | Play record
 | Special case, see below for more info.
 |-
 | 1011
 | Iron door closed
 | 
 |-
 | 1012
 | Wooden door closed
 | 
 |-
 | 1013
 | Wooden trapdoor closed
 | 
 |-
 | 1014
 | Fence gate closed
 | 
 |-
 | 1015
 | Ghast warns
 | 
 |-
 | 1016
 | Ghast shoots
 | 
 |-
 | 1017
 | Enderdragon shoots
 | 
 |-
 | 1018
 | Blaze shoots
 | 
 |-
 | 1019
 | Zombie attacks wood door
 | 
 |-
 | 1020
 | Zombie attacks iron door
 | 
 |-
 | 1021
 | Zombie breaks wood door
 |
 |-
 | 1022
 | Wither breaks block
 |
 |-
 | 1023
 | Wither spawned
 | 
 |-
 | 1024
 | Wither shoots
 |
 |-
 | 1025
 | Bat takes off
 |
 |-
 | 1026
 | Zombie infects
 |
 |-
 | 1027
 | Zombie villager converted
 |
 |-
 | 1028
 | Ender dragon death
 |
 |-
 | 1029
 | Anvil destroyed
 | 
 |-
 | 1030
 | Anvil used
 | 
 |-
 | 1031
 | Anvil landed
 |
 |-
 | 1032
 | Portal travel
 | 
 |-
 | 1033
 | Chorus flower grown
 |
 |-
 | 1034
 | Chorus flower died
 | 
 |-
 | 1035
 | Brewing stand brewed
 |
 |-
 | 1036
 | Iron trapdoor opened
 | 
 |-
 | 1037
 | Iron trapdoor closed
 |
 |-
 | 1038
 | End portal created in overworld
 |
 |-
 | 1039
 | Phantom bites
 |
 |-
 | 1040
 | Zombie converts to drowned
 |
 |-
 | 1041
 | Husk converts to zombie by drowning
 |
 |-
 | 1042
 | Grindstone used
 |
 |-
 | 1043
 | Book page turned
 |
 |-
 |-
 !colspan="3"| Particle
 |-
 | 1500
 | Composter composts
 |
 |-
 | 1501
 | Lava converts block (either water to stone, or removes existing blocks such as torches)
 |
 |-
 | 1502
 | Redstone torch burns out
 |
 |- 
 | 1503
 | Ender eye placed
 |
 |-
 | 2000
 | Spawns 10 smoke particles, e.g. from a fire
 | Direction, see below.
 |-
 | 2001
 | Block break + block break sound
 | Block state, as an index into the global palette.
 |-
 | 2002
 | Splash potion. Particle effect + glass break sound.
 | RGB color as an integer (e.g. 8364543 for #7FA1FF).
 |-
 | 2003
 | Eye of Ender entity break animation — particles and sound
 | 
 |-
 | 2004
 | Mob spawn particle effect: smoke + flames
 | 
 |-
 | 2005
 | Bonemeal particles
 | How many particles to spawn (if set to 0, 15 are spawned).
 |-
 | 2006
 | Dragon breath
 |
 |-
 | 2007
 | Instant splash potion. Particle effect + glass break sound.
 | RGB color as an integer (e.g. 8364543 for #7FA1FF).
 |-
 | 2008
 | Ender dragon destroys block
 |
 |-
 | 2009
 | Wet sponge vaporizes in nether
 |
 |-
 | 3000
 | End gateway spawn
 | 
 |-
 | 3001
 | Enderdragon growl
 |
 |}

Smoke directions:

{| class="wikitable"
 ! ID
 ! Direction
 |-
 | 0
 | Down
 |-
 | 1
 | Up
 |-
 | 2
 | North
 |-
 | 3
 | South
 |-
 | 4
 | West
 |-
 | 5
 | East
 |}

Play record: This is actually a special case within this packet. You can start/stop a record at a specific location. Use a valid {{Minecraft Wiki|Music Discs|Record ID}} to start a record (or overwrite a currently playing one), any other value will stop the record.  See [[Data Generators]] for information on item IDs.

==== Particle ====

Displays the named particle

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="11"| 0x22
 |rowspan="11"| Play
 |rowspan="11"| Client
 | Particle ID
 | Int
 | The particle ID listed in [[#Particle|the particle data type]].
 |-
 | Long Distance
 | Boolean
 | If true, particle distance increases from 256 to 65536.
 |-
 | X
 | Double
 | X position of the particle.
 |-
 | Y
 | Double
 | Y position of the particle.
 |-
 | Z
 | Double
 | Z position of the particle.
 |-
 | Offset X
 | Float
 | This is added to the X position after being multiplied by <code>random.nextGaussian()</code>.
 |-
 | Offset Y
 | Float
 | This is added to the Y position after being multiplied by <code>random.nextGaussian()</code>.
 |-
 | Offset Z
 | Float
 | This is added to the Z position after being multiplied by <code>random.nextGaussian()</code>.
 |-
 | Particle Data
 | Float
 | The data of each particle.
 |-
 | Particle Count
 | Int
 | The number of particles to create.
 |-
 | Data
 | Varies
 | The variable data listed in [[#Particle|the particle data type]].
 |}

==== Update Light ====

Updates light levels for a chunk.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="11"| 0x23
 |rowspan="11"| Play
 |rowspan="11"| Client
 |colspan="2"| Chunk X
 |colspan="2"| VarInt
 | Chunk coordinate (block coordinate divided by 16, rounded down). 
 |-
 |colspan="2"| Chunk Z
 |colspan="2"| VarInt
 | Chunk coordinate (block coordinate divided by 16, rounded down).
 |-
 |colspan="2"| Trust Edges
 |colspan="2"| Boolean
 | If edges should be trusted for light updates.
 |-
 |colspan="2"| Sky Light Mask
 |colspan="2"| VarInt
 | Mask containing 18 bits, with the lowest bit corresponding to chunk section -1 (in the void, y=-16 to y=-1) and the highest bit for chunk section 16 (above the world, y=256 to y=271).
 |-
 |colspan="2"| Block Light Mask
 |colspan="2"| VarInt
 | Mask containing 18 bits, with the same order as sky light.
 |-
 |colspan="2"| Empty Sky Light Mask
 |colspan="2"| VarInt
 | Mask containing 18 bits, which indicates sections that have 0 for all their sky light values.  If a section is set in both this mask and the main sky light mask, it is ignored for this mask and it is included in the sky light arrays (the notchian server does not create such masks).  If it is only set in this mask, it is not included in the sky light arrays.
 |-
 |colspan="2"| Empty Block Light Mask
 |colspan="2"| VarInt
 | Mask containing 18 bits, which indicates sections that have 0 for all their block light values.  If a section is set in both this mask and the main block light mask, it is ignored for this mask and it is included in the block light arrays (the notchian server does not create such masks).  If it is only set in this mask, it is not included in the block light arrays.
 |-
 |rowspan="2"| Sky Light arrays
 | Length
 |rowspan="2"| Array
 | VarInt
 | Length of the following array in bytes (always 2048).
 |-
 | Sky Light array
 | Array of 2048 bytes
 | There is 1 array for each bit set to true in the sky light mask, starting with the lowest value.  Half a byte per light value.
 |-
 |rowspan="2"| Block Light arrays
 | Length
 |rowspan="2"| Array
 | VarInt
 | Length of the following array in bytes (always 2048).
 |-
 | Block Light array
 | Array of 2048 bytes
 | There is 1 array for each bit set to true in the block light mask, starting with the lowest value.  Half a byte per light value.
 |}

Individual block or sky light arrays are given for each block with increasing x coordinates, within rows of increasing z coordinates, within layers of increasing y coordinates. Even-indexed items (those with an even x coordinate, starting at 0) are packed into the low bits, odd-indexed into the high bits.

For the arrays: there are 18 of each, 18 block light arrays and 18 sky light arrays. Each array is in the format described, 2048 as a VarInt, followed by 2048 bytes which describe 4096 items. The update masks use their 18 least significant bits to indicate whether or not the packet will include an array at that index. For example: if all bits except for the least significant bit are set to 1, and the least significant bit is set to 0, then all arrays except for array #0 are included (for a total of 17 arrays or 17 * 2048 bytes).

==== Join Game ====

See [[Protocol Encryption]] for information on logging in.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="16"| 0x24
 |rowspan="16"| Play
 |rowspan="16"| Client
 | Entity ID
 | Int
 | The player's Entity ID (EID).
 |-
 | Is hardcore
 | Boolean
 |
 |-
 | Gamemode
 | Unsigned Byte
 | 0: Survival, 1: Creative, 2: Adventure, 3: Spectator.
 |-
 | Previous Gamemode
 | Byte
 | 0: survival, 1: creative, 2: adventure, 3: spectator. The hardcore flag is not included. The previous gamemode. Defaults to -1 if there is no previous gamemode. (More information needed)
 |-
 | World Count
 | VarInt
 | Size of the following array.
 |-
 | World Names
 | Array of Identifier
 | Identifiers for all worlds on the server.
 |-
 | Dimension Codec
 | [[NBT|NBT Tag Compound]]
 | The full extent of these is still unknown, but the tag represents a dimension and biome registry. See below for the vanilla default.
 |-
 | Dimension
 | [[NBT|NBT Tag Compound]]
 | Valid dimensions are defined per dimension registry sent before this. The structure of this tag is a dimension type (see below).
 |- 
 | World Name
 | Identifier
 | Name of the world being spawned into.
 |-
 | Hashed seed
 | Long
 | First 8 bytes of the SHA-256 hash of the world's seed. Used client side for biome noise
 |-
 | Max Players
 | VarInt
 | Was once used by the client to draw the player list, but now is ignored.
 |-
 | View Distance
 | VarInt
 | Render distance (2-32).
 |-
 | Reduced Debug Info
 | Boolean
 | If true, a Notchian client shows reduced information on the {{Minecraft Wiki|debug screen}}.  For servers in development, this should almost always be false.
 |-
 | Enable respawn screen
 | Boolean
 | Set to false when the doImmediateRespawn gamerule is true.
 |-
 | Is Debug
 | Boolean
 | True if the world is a {{Minecraft Wiki|debug mode}} world; debug mode worlds cannot be modified and have predefined blocks.
 |-
 | Is Flat
 | Boolean
 | True if the world is a {{Minecraft Wiki|superflat}} world; flat worlds have different void fog and a horizon at y=0 instead of y=63.
 |}


The '''Dimension Codec''' NBT Tag Compound ([https://gist.githubusercontent.com/aramperes/44e2beefac9fe966177f2f28dd0136ab/raw/fedb31c32e27265fb916a68ad476470fc65631da/1-dimension_codec.snbt Default value in SNBT]) includes two registries: "minecraft:dimension_type" and "minecraft:worldgen/biome".

{| class="wikitable"
 !Name
 !Type
 !style="width: 250px;" colspan="2"| Notes
 |-
 | minecraft:dimension_type
 | TAG_Compound
 | The dimension type registry (see below).
 |-
 | minecraft:worldgen/biome
 | TAG_Compound
 | The biome registry (see below).
 |}

Dimension type registry:

{| class="wikitable"
 !Name
 !Type
 !style="width: 250px;" colspan="2"| Notes
 |-
 | type
 | TAG_String
 | The name of the registry. Always "minecraft:dimension_type".
 |-
 | value
 | TAG_List
 | List of dimension types registry entries (see below).
 |}

Dimension type registry entry:

{| class="wikitable"
 !Name
 !Type
 !style="width: 250px;" colspan="2"| Notes
 |-
 | name
 | TAG_String
 | The name of the dimension type (for example, "minecraft:overworld").
 |-
 | id
 | TAG_Int
 | The protocol ID of the dimension (matches the index of the element in the registry list).
 |-
 | element
 | TAG_Compound
 | The dimension type (see below).
 |}

Dimension type:

{| class="wikitable"
 ! Name
 ! Type
 !style="width: 250px;" colspan="2"| Meaning
 ! Values
 |-
 | piglin_safe
 | TAG_Byte
 |colspan="2"| Whether piglins shake and transform to zombified piglins.
 | 1: true, 0: false.
 |-
 | natural
 | TAG_Byte
 |colspan="2"| When false, compasses spin randomly. When true, nether portals can spawn zombified piglins.
 | 1: true, 0: false.
 |-
 | ambient_light
 | TAG_Float
 |colspan="2"| How much light the dimension has.
 | 0.0 to 1.0.
 |-
 | fixed_time
 | Optional TAG_Long
 |colspan="2"| If set, the time of the day is the specified value.
 | If set, 0 to 24000.
 |-
 | infiniburn
 | TAG_String
 |colspan="2"| A resource location defining what block tag to use for infiniburn.
 | "" or minecraft resource "minecraft:...".
 |-
 | respawn_anchor_works
 | TAG_Byte
 |colspan="2"| Whether players can charge and use respawn anchors.
 | 1: true, 0: false.
 |-
 | has_skylight
 | TAG_Byte
 |colspan="2"| Whether the dimension has skylight access or not.
 | 1: true, 0: false.
 |-
 | bed_works
 | TAG_Byte
 |colspan="2"| Whether players can use a bed to sleep.
 | 1: true, 0: false.
 |-
 | effects
 | TAG_String
 |colspan="2"| ?
 | "minecraft:overworld", "minecraft:the_nether", "minecraft:the_end" or something else.
 |-
 | has_raids
 | TAG_Byte
 |colspan="2"| Whether players with the Bad Omen effect can cause a raid.
 | 1: true, 0: false.
 |-
 | logical_height
 | TAG_Int
 |colspan="2"| The maximum height to which chorus fruits and nether portals can bring players within this dimension.
 | 0-256.
 |-
 | coordinate_scale
 | TAG_Float
 |colspan="2"| The multiplier applied to coordinates when traveling to the dimension.
 | 1: true, 0: false.
 |-
 | ultrawarm
 | TAG_Byte
 |colspan="2"| Whether the dimensions behaves like the nether (water evaporates and sponges dry) or not. Also causes lava to spread thinner.
 | 1: true, 0: false.
 |-
 | has_ceiling
 | TAG_Byte
 |colspan="2"| Whether the dimension has a bedrock ceiling or not. When true, causes lava to spread faster.
 | 1: true, 0: false.
 |}

Biome registry:

{| class="wikitable"
 !Name
 !Type
 !style="width: 250px;" colspan="2"| Notes
 |-
 | type
 | TAG_String
 | The name of the registry. Always "minecraft:worldgen/biome".
 |-
 | value
 | TAG_List
 | List of biome registry entries (see below).
 |}

Biome registry entry:

{| class="wikitable"
 !Name
 !Type
 !style="width: 250px;" colspan="2"| Notes
 |-
 | name
 | TAG_String
 | The name of the biome (for example, "minecraft:ocean").
 |-
 | id
 | TAG_Int
 | The protocol ID of the biome (matches the index of the element in the registry list).
 |-
 | element
 | TAG_Compound
 | The biome properties (see below).
 |}

Biome properties:

{| class="wikitable"
 !colspan="2"|Name
 !colspan="2"|Type
 !style="width: 250px;" colspan="2"| Meaning
 !colspan="2"|Values
 |-
 |colspan="2"|precipitation
 |colspan="2"|TAG_String
 |colspan="2"| The type of precipitation in the biome.
 |colspan="2"|"rain", "snow", or "none".
 |-
 |colspan="2"| depth
 |colspan="2"| TAG_Float
 |colspan="2"| The depth factor of the biome.
 |colspan="2"| The default values vary between 1.5 and -1.8.
 |-
 |colspan="2"| temperature
 |colspan="2"| TAG_Float
 |colspan="2"| The temperature factor of the biome.
 |colspan="2"| The default values vary between 2.0 and -0.5.
 |-
 |colspan="2"| scale
 |colspan="2"| TAG_Float
 |colspan="2"| ?
 |colspan="2"| The default values vary between 1.225 and 0.0.
 |-
 |colspan="2"| downfall
 |colspan="2"| TAG_Float
 |colspan="2"| ?
 |colspan="2"| The default values vary between 1.0 and 0.0.
 |-
 |colspan="2"| category
 |colspan="2"| TAG_String
 |colspan="2"| The category of the biome.
 |colspan="2"| Known values are "ocean", "plains", "desert", "forest", "extreme_hills", "taiga", "swamp", "river", "nether", "the_end", "icy", "mushroom", "beach", "jungle", "mesa", "savanna", and "none". 
 |-
 |colspan="2"| temperature_modifier
 |colspan="2"| Optional TAG_String
 |colspan="2"| ?
 |colspan="2"| The only known value is "frozen".
 |-
 |rowspan="11"| effects
 | sky_color
 |rowspan="11"| TAG_Compound
 | TAG_Int
 |colspan="2"| The color of the sky.
 | Example: 8364543, which is #7FA1FF in RGB.
 |-
 | water_fog_color
 | TAG_Int
 |colspan="2"| Possibly the tint color when swimming.
 | Example: 8364543, which is #7FA1FF in RGB.
 |-
 | fog_color
 | TAG_Int
 |colspan="2"| Possibly the color of the fog effect when looking past the view distance.
 | Example: 8364543, which is #7FA1FF in RGB.
 |-
 | water_color
 | TAG_Int
 |colspan="2"| The tint color of the water blocks.
 | Example: 8364543, which is #7FA1FF in RGB.
 |-
 | foliage_color
 | Optional TAG_Int
 |colspan="2"| The tint color of the grass.
 | Example: 8364543, which is #7FA1FF in RGB.
 |- 
 | grass_color
 | Optional TAG_Int
 | colspan="2"| ?
 | Example: 8364543, which is #7FA1FF in RGB.
 |-
 | grass_color_modifier
 | Optional TAG_String
 |colspan="2"| Unknown, likely affects foliage color.
 | If set, known values are "swamp" and "dark_forest".
 |-
 | music
 | Optional TAG_Compound
 |colspan="2"| Music properties for the biome.
 | If present, contains the fields: replace_current_music (TAG_Byte), sound (TAG_String), max_delay (TAG_Int), min_delay (TAG_Int).
 |-
 | ambient_sound
 | Optional TAG_String
 |colspan="2"| Ambient soundtrack.
 | If present, the ID of a soundtrack. Example: "minecraft:ambient.basalt_deltas.loop".
 |-
 | additions_sound
 | Optional TAG_Compound
 |colspan="2"| Additional ambient sound that plays randomly.
 | If present, contains the fields: sound (TAG_String), tick_chance (TAG_Double).
 |-
 | mood_sound
 | Optional TAG_Compound
 |colspan="2"| Additional ambient sound that plays at an interval.
 | If present, contains the fields: sound (TAG_String), tick_delay (TAG_Int), offset (TAG_Double), block_search_extent (TAG_Int).
 |-
 | particle
 | Optional TAG_Compound
 |colspan="2"| Particles that appear randomly in the biome.
 | If present, contains the fields: probability (TAG_Float), options (TAG_Compound). The "options" compound contains the field "type" (TAG_String), which identifies the particle type.
 |}

==== Map Data ====

Updates a rectangular area on a {{Minecraft Wiki|map}} item.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="17"| 0x25
 |rowspan="17"| Play
 |rowspan="17"| Client
 |colspan="2"| Map ID
 |colspan="2"| VarInt
 | Map ID of the map being modified.
 |-
 |colspan="2"| Scale
 |colspan="2"| Byte
 | From 0 for a fully zoomed-in map (1x1 blocks per pixel) to 4 for a fully zoomed-out map (16x16 blocks per pixel).
 |- 
 |colspan="2"| Tracking Position
 |colspan="2"| Boolean
 | Specifies whether player and item frame icons are shown.
 |-
 |colspan="2"| Locked
 |colspan="2"| Boolean
 | True if the map has been locked in a cartography table.
 |- 
 |colspan="2"| Icon Count
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="6"| Icon
 | Type
 |rowspan="6"| Array
 | VarInt enum
 | See below.
 |-
 | X
 | Byte
 | Map coordinates: -128 for furthest left, +127 for furthest right.
 |-
 | Z
 | Byte
 | Map coordinates: -128 for highest, +127 for lowest.
 |-
 | Direction
 | Byte
 | 0-15.
 |-
 | Has Display Name
 | Boolean
 |
 |-
 | Display Name
 | Optional [[Chat]]
 | Only present if previous Boolean is true.
 |- 
 |colspan="2"| Columns
 |colspan="2"| Unsigned Byte
 | Number of columns updated.
 |-
 |colspan="2"| Rows
 |colspan="2"| Optional Unsigned Byte
 | Only if Columns is more than 0; number of rows updated.
 |-
 |colspan="2"| X
 |colspan="2"| Optional Byte
 | Only if Columns is more than 0; x offset of the westernmost column.
 |-
 |colspan="2"| Z
 |colspan="2"| Optional Byte
 | Only if Columns is more than 0; z offset of the northernmost row.
 |-
 |colspan="2"| Length
 |colspan="2"| Optional VarInt
 | Only if Columns is more than 0; length of the following array.
 |-
 |colspan="2"| Data
 |colspan="2"| Optional Array of Unsigned Byte
 | Only if Columns is more than 0; see {{Minecraft Wiki|Map item format}}.
 |}

For icons, a direction of 0 is a vertical icon and increments by 22.5&deg; (360/16).

Types are based off of rows and columns in <code>map_icons.png</code>:

{| class="wikitable"
 |-
 ! Icon type
 ! Result
 |-
 | 0
 | White arrow (players)
 |-
 | 1
 | Green arrow (item frames)
 |-
 | 2
 | Red arrow
 |-
 | 3
 | Blue arrow
 |-
 | 4
 | White cross
 |-
 | 5
 | Red pointer
 |-
 | 6
 | White circle (off-map players)
 |-
 | 7
 | Small white circle (far-off-map players)
 |-
 | 8
 | Mansion
 |-
 | 9
 | Temple
 |-
 | 10
 | White Banner
 |-
 | 11
 | Orange Banner
 |-
 | 12
 | Magenta Banner
 |-
 | 13
 | Light Blue Banner
 |-
 | 14
 | Yellow Banner
 |-
 | 15
 | Lime Banner
 |-
 | 16
 | Pink Banner
 |-
 | 17
 | Gray Banner
 |-
 | 18
 | Light Gray Banner
 |-
 | 19
 | Cyan Banner
 |-
 | 20
 | Purple Banner
 |-
 | 21
 | Blue Banner
 |-
 | 22
 | Brown Banner
 |-
 | 23
 | Green Banner
 |-
 | 24
 | Red Banner
 |-
 | 25
 | Black Banner
 |-
 | 26
 | Treasure marker
 |}

==== Trade List ====

The list of trades a villager NPC is offering.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="18"| 0x26
 |rowspan="18"| Play
 |rowspan="18"| Client
 |-
 | colspan="2" | Window ID
 | colspan="2" | VarInt
 | The ID of the window that is open; this is an int rather than a byte.
 |-
 | colspan="2" | Size
 | colspan="2" | Byte
 | The number of trades in the following array.
 |-
 | rowspan="11" | Trades
 | Input item 1
 | rowspan="11" | Array
 | [[Slot]]
 | The first item the player has to supply for this villager trade. The count of the item stack is the default "price" of this trade.
 |-
 | Output item
 | [[Slot]]
 | The item the player will receive from this villager trade.
 |-
 | Has second item
 | Boolean
 | Whether there is a second item.
 |-
 | Input item 2
 | Optional [[Slot]]
 | The second item the player has to supply for this villager trade; only present if has a second item is true.
 |-
 | Trade disabled
 | Boolean
 | True if the trade is disabled; false if the trade is enabled.
 |-
 | Number of trade uses
 | Integer
 | Number of times the trade has been used so far. If equal to the maximum number of trades, the client will display a red X.
 |-
 | Maximum number of trade uses
 | Integer
 | Number of times this trade can be used before it's exhausted.
 |-
 | XP
 | Integer
 | Amount of XP both the player and the villager will earn each time the trade is used.
 |-
 | Special Price
 | Integer
 | Can be zero or negative. The number is added to the price when an item is discounted due to player reputation or other effects.
 |-
 | Price Multiplier
 | Float
 | Can be low (0.05) or high (0.2). Determines how much demand, player reputation, and temporary effects will adjust the price.
 |-
 | Demand
 | Integer
 | Can be zero or a positive number. Causes the price to increase when a trade is used repeatedly.
 |-
 |colspan="2"| Villager level
 |colspan="2"| VarInt
 | Appears on the trade GUI; meaning comes from the translation key <code>merchant.level.</code> + level.
1: Novice, 2: Apprentice, 3: Journeyman, 4: Expert, 5: Master.
 |-
 |colspan="2"| Experience
 |colspan="2"| VarInt
 | Total experience for this villager (always 0 for the wandering trader).
 |-
 |colspan="2"| Is regular villager
 |colspan="2"| Boolean
 | True if this is a regular villager; false for the wandering trader.  When false, hides the villager level and some other GUI elements.
 |-
 |colspan="2"| Can restock
 |colspan="2"| Boolean
 | True for regular villagers and false for the wandering trader. If true, the "Villagers restock up to two times per day." message is displayed when hovering over disabled trades.
 |}

Modifiers can increase or decrease the number of items for the first input slot. The second input slot and the output slot never change the nubmer of items. The number of items may never be less than 1, and never more than the stack size. If special price and demand are both zero, only the default price is displayed. If either is non-zero, then the adjusted price is displayed next to the crossed-out default price. The adjusted prices is calculated as follows:

Adjusted price = default price + floor(default price x multiplier x demand) + special price

[[File:1.14-merchant-slots.png|thumb|The merchant UI, for reference]]
{{-}}

==== Entity Position ====

This packet is sent by the server when an entity moves less then 8 blocks; if an entity moves more than 8 blocks [[#Entity Teleport|Entity Teleport]] should be sent instead.

This packet allows at most 8 blocks movement in any direction, because short range is from -32768 to 32767. And <code>32768 / (128 * 32)</code> = 8.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x27
 |rowspan="5"| Play
 |rowspan="5"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Delta X
 | Short
 | Change in X position as <code>(currentX * 32 - prevX * 32) * 128</code>.
 |-
 | Delta Y
 | Short
 | Change in Y position as <code>(currentY * 32 - prevY * 32) * 128</code>.
 |-
 | Delta Z
 | Short
 | Change in Z position as <code>(currentZ * 32 - prevZ * 32) * 128</code>.
 |-
 | On Ground
 | Boolean
 | 
 |}

==== Entity Position and Rotation ====

This packet is sent by the server when an entity rotates and moves. Since a short range is limited from -32768 to 32767, and movement is offset of fixed-point numbers, this packet allows at most 8 blocks movement in any direction. (<code>-32768 / (32 * 128) == -8</code>)

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x28
 |rowspan="7"| Play
 |rowspan="7"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Delta X
 | Short
 | Change in X position as <code>(currentX * 32 - prevX * 32) * 128</code>.
 |-
 | Delta Y
 | Short
 | Change in Y position as <code>(currentY * 32 - prevY * 32) * 128</code>.
 |-
 | Delta Z
 | Short
 | Change in Z position as <code>(currentZ * 32 - prevZ * 32) * 128</code>.
 |-
 | Yaw
 | Angle
 | New angle, not a delta.
 |-
 | Pitch
 | Angle
 | New angle, not a delta.
 |-
 | On Ground
 | Boolean
 | 
 |}

==== Entity Rotation ====

This packet is sent by the server when an entity rotates.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x29
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Yaw
 | Angle
 | New angle, not a delta.
 |-
 | Pitch
 | Angle
 | New angle, not a delta.
 |-
 | On Ground
 | Boolean
 | 
 |}

==== Entity Movement ====

This packet may be used to initialize an entity.

For player entities, either this packet or any move/look packet is sent every game tick. So the meaning of this packet is basically that the entity did not move/look since the last such packet.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2A
 | Play
 | Client
 | Entity ID
 | VarInt
 | 
 |}

==== Vehicle Move (clientbound) ====

Note that all fields use absolute positioning and do not allow for relative positioning.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x2B
 |rowspan="5"| Play
 |rowspan="5"| Client
 | X
 | Double
 | Absolute position (X coordinate).
 |-
 | Y
 | Double
 | Absolute position (Y coordinate).
 |-
 | Z
 | Double
 | Absolute position (Z coordinate).
 |-
 | Yaw
 | Float
 | Absolute rotation on the vertical axis, in degrees.
 |-
 | Pitch
 | Float
 | Absolute rotation on the horizontal axis, in degrees.
 |}

==== Open Book ====

Sent when a player right clicks with a signed book. This tells the client to open the book GUI.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2C
 | Play
 | Client
 | Hand
 | VarInt enum
 | 0: Main hand, 1: Off hand .
 |}

==== Open Window ====

This is sent to the client when it should open an inventory, such as a chest, workbench, or furnace. This message is not sent anywhere for clients opening their own inventory.  For horses, use [[#Open Horse Window|Open Horse Window]].

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x2D
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Window ID
 | VarInt
 | A unique id number for the window to be displayed. Notchian server implementation is a counter, starting at 1.
 |-
 | Window Type
 | VarInt
 | The window type to use for display. Contained in the <code>minecraft:menu</code> registry; see [[Inventory]] for the different values.
 |-
 | Window Title
 | [[Chat]]
 | The title of the window.
 |}

==== Open Sign Editor ====

Sent when the client has placed a sign and is allowed to send [[#Update Sign|Update Sign]].  There must already be a sign at the given location (which the client does not do automatically) - send a [[#Block Change|Block Change]] first.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2E
 | Play
 | Client
 | Location
 | Position
 | 
 |}

==== Craft Recipe Response ====

Response to the serverbound packet ([[#Craft Recipe Request|Craft Recipe Request]]), with the same recipe ID.  Appears to be used to notify the UI.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x2F
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Window ID
 | Byte
 |
 |-
 | Recipe
 | Identifier
 | A recipe ID.
 |}

==== Player Abilities (clientbound) ====

The latter 2 floats are used to indicate the field of view and flying speed respectively, while the first byte is used to determine the value of 4 booleans.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x30
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Flags
 | Byte
 | Bit field, see below.
 |-
 | Flying Speed
 | Float
 | 0.05 by default.
 |-
 | Field of View Modifier
 | Float
 | Modifies the field of view, like a speed potion. A Notchian server will use the same value as the movement speed sent in the [[#Entity_Properties|Entity Properties]] packet, which defaults to 0.1 for players.
 |}

About the flags:

{| class="wikitable"
 |-
 ! Field
 ! Bit
 |-
 | Invulnerable
 | 0x01
 |-
 | Flying
 | 0x02
 |-
 | Allow Flying
 | 0x04
 |-
 | Creative Mode (Instant Break)
 | 0x08
 |}

==== Combat Event ====

Originally used for metadata for twitch streaming circa 1.8.  Now only used to display the game over screen (with enter combat and end combat completely ignored by the Notchain client)

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="8"| 0x31
 |rowspan="8"| Play
 |rowspan="8"| Client
 |colspan="2"| Event
 | VarInt Enum
 | Determines the layout of the remaining packet.
 |-
 ! Event
 ! Field Name
 ! 
 ! 
 |-
 | 0: enter combat
 | ''no fields''
 | ''no fields''
 | 
 |-
 |rowspan="2"| 1: end combat
 | Duration
 | VarInt
 | Length of the combat in ticks.
 |-
 | Entity ID
 | Int
 | ID of the primary opponent of the ended combat, or -1 if there is no obvious primary opponent.
 |-
 |rowspan="3"| 2: entity dead
 | Player ID
 | VarInt
 | Entity ID of the player that died (should match the client's entity ID).
 |-
 | Entity ID
 | Int
 | The killing entity's ID, or -1 if there is no obvious killer.
 |-
 | Message
 | [[Chat]]
 | The death message.
 |}

==== Player Info ====

Sent by the server to update the user list (<tab> in the client).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="4"| Field Name
 !colspan="3"| Field Type
 ! Notes
 |-
 |rowspan="19"| 0x32
 |rowspan="19"| Play
 |rowspan="19"| Client
 |colspan="4"| Action
 |colspan="3"| VarInt
 | Determines the rest of the Player format after the UUID.
 |-
 |colspan="4"| Number Of Players
 |colspan="3"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="17"| Player
 |colspan="3"| UUID
 |rowspan="17"| Array
 |colspan="2"| UUID
 | 
 |-
 ! Action
 !colspan="2"| Field Name
 !colspan="2"| 
 ! 
 |-
 |rowspan="10"| 0: add player
 |colspan="2"| Name
 |colspan="2"| String (16)
 | 
 |-
 |colspan="2"| Number Of Properties
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="4"| Property
 | Name
 |rowspan="4"| Array
 | String (32767)
 | 
 |-
 | Value
 | String (32767)
 | 
 |- 
 | Is Signed
 | Boolean
 | 
 |-
 | Signature
 | Optional String (32767)
 | Only if Is Signed is true.
 |-
 |colspan="2"| Gamemode
 |colspan="2"| VarInt
 | 
 |-
 |colspan="2"| Ping
 |colspan="2"| VarInt
 | Measured in milliseconds.
 |-
 |colspan="2"| Has Display Name
 |colspan="2"| Boolean
 | 
 |-
 |colspan="2"| Display Name
 |colspan="2"| Optional [[Chat]]
 | Only if Has Display Name is true.
 |-
 | 1: update gamemode
 |colspan="2"| Gamemode
 |colspan="2"| VarInt
 | 
 |- 
 | 2: update latency
 |colspan="2"| Ping
 |colspan="2"| VarInt
 | Measured in milliseconds.
 |-
 |rowspan="2"| 3: update display name
 |colspan="2"| Has Display Name
 |colspan="2"| Boolean
 | 
 |-
 |colspan="2"| Display Name
 |colspan="2"| Optional [[Chat]]
 | Only send if Has Display Name is true.
 |-
 | 4: remove player
 |colspan="2"| ''no fields''
 |colspan="2"| ''no fields''
 | 
 |}

The Property field looks as in the response of [[Mojang API#UUID -> Profile + Skin/Cape]], except of course using the protocol format instead of JSON. That is, each player will usually have one property with Name “textures” and Value being a base64-encoded JSON string as documented at [[Mojang API#UUID -> Profile + Skin/Cape]]. An empty properties array is also acceptable, and will cause clients to display the player with one of the two default skins depending on UUID.

Ping values correspond with icons in the following way:
* A ping that negative (i.e. not known to the server yet) will result in the no connection icon.
* A ping under 150 milliseconds will result in 5 bars
* A ping under 300 milliseconds will result in 4 bars
* A ping under 600 milliseconds will result in 3 bars
* A ping under 1000 milliseconds (1 second) will result in 2 bars
* A ping greater than or equal to 1 second will result in 1 bar.

==== Face Player ====

Used to rotate the client player to face the given location or entity (for <code>/teleport [<targets>] <x> <y> <z> facing</code>).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="8"| 0x33
 |rowspan="8"| Play
 |rowspan="8"| Client
 |-
 | Feet/eyes
 | VarInt enum
 | Values are feet=0, eyes=1.  If set to eyes, aims using the head position; otherwise aims using the feet position.
 |-
 | Target x
 | Double
 | x coordinate of the point to face towards.
 |-
 | Target y
 | Double
 | y coordinate of the point to face towards.
 |-
 | Target z
 | Double
 | z coordinate of the point to face towards.
 |-
 | Is entity
 | Boolean
 | If true, additional information about an entity is provided.
 |-
 | Entity ID
 | Optional VarInt
 | Only if is entity is true &mdash; the entity to face towards.
 |-
 | Entity feet/eyes
 | Optional VarInt enum
 | Whether to look at the entity's eyes or feet.  Same values and meanings as before, just for the entity's head/feet.
 |}

If the entity given by entity ID cannot be found, this packet should be treated as if is entity was false.

==== Player Position And Look (clientbound) ==== 

Updates the player's position on the server. This packet will also close the “Downloading Terrain” screen when joining/respawning.

If the distance between the last known position of the player on the server and the new position set by this packet is greater than 100 meters, the client will be kicked for “You moved too quickly :( (Hacking?)”.

Also if the fixed-point number of X or Z is set greater than <code>3.2E7D</code> the client will be kicked for “Illegal position”.

Yaw is measured in degrees, and does not follow classical trigonometry rules. The unit circle of yaw on the XZ-plane starts at (0, 1) and turns counterclockwise, with 90 at (-1, 0), 180 at (0, -1) and 270 at (1, 0). Additionally, yaw is not clamped to between 0 and 360 degrees; any number is valid, including negative numbers and numbers greater than 360.

Pitch is measured in degrees, where 0 is looking straight ahead, -90 is looking straight up, and 90 is looking straight down.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x34
 |rowspan="7"| Play
 |rowspan="7"| Client
 | X
 | Double
 | Absolute or relative position, depending on Flags.
 |-
 | Y
 | Double
 | Absolute or relative position, depending on Flags.
 |-
 | Z
 | Double
 | Absolute or relative position, depending on Flags.
 |-
 | Yaw
 | Float
 | Absolute or relative rotation on the X axis, in degrees.
 |-
 | Pitch
 | Float
 | Absolute or relative rotation on the Y axis, in degrees.
 |-
 | Flags
 | Byte
 | Bit field, see below.
 |-
 | Teleport ID
 | VarInt
 | Client should confirm this packet with [[#Teleport Confirm|Teleport Confirm]] containing the same Teleport ID.
 |}

About the Flags field:

 <Dinnerbone> It's a bitfield, X/Y/Z/Y_ROT/X_ROT. If X is set, the x value is relative and not absolute.

{| class="wikitable"
 |-
 ! Field
 ! Bit
 |-
 | X
 | 0x01
 |-
 | Y
 | 0x02
 |-
 | Z
 | 0x04
 |-
 | Y_ROT
 | 0x08
 |-
 | X_ROT
 | 0x10
 |}

==== Unlock Recipes ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="14"| 0x35
 |rowspan="14"| Play
 |rowspan="14"| Client
 |-
 | Action
 | VarInt
 | 0: init, 1: add, 2: remove.
 |-
 | Crafting Recipe Book Open
 | Boolean
 | If true, then the crafting recipe book will be open when the player opens its inventory.
 |-
 | Crafting Recipe Book Filter Active
 | Boolean
 | If true, then the filtering option is active when the players opens its inventory.
 |-
 | Smelting Recipe Book Open
 | Boolean
 | If true, then the smelting recipe book will be open when the player opens its inventory.
 |-
 | Smelting Recipe Book Filter Active
 | Boolean
 | If true, then the filtering option is active when the players opens its inventory.
 |-
 | Blast Furnace Recipe Book Open
 | Boolean
 | If true, then the blast furnace recipe book will be open when the player opens its inventory.
 |-
 | Blast Furnace Recipe Book Filter Active
 | Boolean
 | If true, then the filtering option is active when the players opens its inventory.
 |-
 | Smoker Recipe Book Open
 | Boolean
 | If true, then the smoker recipe book will be open when the player opens its inventory.
 |-
 | Smoker Recipe Book Filter Active
 | Boolean
 | If true, then the filtering option is active when the players opens its inventory.
 |-
 | Array size 1
 | VarInt
 | Number of elements in the following array.
 |-
 | Recipe IDs
 | Array of Identifier
 |
 |-
 | Array size 2
 | Optional VarInt
 | Number of elements in the following array, only present if mode is 0 (init).
 |-
 | Recipe IDs
 | Optional Array of Identifier
 | Only present if mode is 0 (init)
 |}
Action:
* 0 (init) = All the recipes in list 1 will be tagged as displayed, and all the recipes in list 2 will be added to the recipe book. Recipes that aren't tagged will be shown in the notification.
* 1 (add) = All the recipes in the list are added to the recipe book and their icons will be shown in the notification.
* 2 (remove) = Remove all the recipes in the list. This allows them to be re-displayed when they are re-added.

==== Destroy Entities ====

Sent by the server when a list of entities is to be destroyed on the client.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x36
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Entity IDs
 | Array of VarInt
 | The list of entities of destroy.
 |}

==== Remove Entity Effect ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x37
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Effect ID
 | Byte
 | See {{Minecraft Wiki|Status effect#Effect IDs|this table}}.
 |}

==== Resource Pack Send ==== 

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x38
 |rowspan="2"| Play
 |rowspan="2"| Client
 | URL
 | String (32767)
 | The URL to the resource pack.
 |-
 | Hash
 | String (40)
 | A 40 character hexadecimal and lowercase [[wikipedia:SHA-1|SHA-1]] hash of the resource pack file. (must be lower case in order to work)<br />If it's not a 40 character hexadecimal string, the client will not use it for hash verification and likely waste bandwidth — but it will still treat it as a unique id.
 |}

==== Respawn ====

To change the player's dimension (overworld/nether/end), send them a respawn packet with the appropriate dimension, followed by prechunks/chunks for the new dimension, and finally a position and look packet. You do not need to unload chunks, the client will do it automatically.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="8"| 0x39
 |rowspan="8"| Play
 |rowspan="8"| Client
 | Dimension
 | [[NBT|NBT Tag Compound]]
 | |Valid dimensions are defined per dimension registry sent in [[#Join Game|Join Game]]
 |-
 | World Name
 | Identifier
 | Name of the world being spawned into
 |-
 | Hashed seed
 | Long
 | First 8 bytes of the SHA-256 hash of the world's seed. Used client side for biome noise
 |- 
 | Gamemode
 | Unsigned Byte
 | 0: survival, 1: creative, 2: adventure, 3: spectator. The hardcore flag is not included
 |- 
 | Previous Gamemode
 | Unsigned Byte
 | 0: survival, 1: creative, 2: adventure, 3: spectator. The hardcore flag is not included. The previous gamemode. (More information needed)
 |-
 | Is Debug
 | Boolean
 | True if the world is a {{Minecraft Wiki|debug mode}} world; debug mode worlds cannot be modified and have predefined blocks.
 |-
 | Is Flat
 | Boolean
 | True if the world is a {{Minecraft Wiki|superflat}} world; flat worlds have different void fog and a horizon at y=0 instead of y=63.
 |-
 | Copy metadata
 | Boolean
 | If false, metadata is reset on the respawned player entity.  Set to true for dimension changes (including the dimension change triggered by sending client status perform respawn to exit the end poem/credits), and false for normal respawns.
 |}

{{Need Info|Does the new World Name field resolve this same-dimension issue?}}

{{Warning2|Avoid changing player's dimension to same dimension they were already in unless they are dead. If you change the dimension to one they are already in, weird bugs can occur, such as the player being unable to attack other players in new world (until they die and respawn).

If you must respawn a player in the same dimension without killing them, send two respawn packets, one to a different world and then another to the world you want. You do not need to complete the first respawn; it only matters that you send two packets.}}

==== Entity Head Look ====

Changes the direction an entity's head is facing.

While sending the Entity Look packet changes the vertical rotation of the head, sending this packet appears to be necessary to rotate the head horizontally.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x3A
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Head Yaw
 | Angle
 | New angle, not a delta.
 |}
 
==== Multi Block Change ====

Fired whenever 2 or more blocks are changed within the same chunk on the same tick.

{{Warning|Changing blocks in chunks not loaded by the client is unsafe (see note on [[#Block Change|Block Change]]).}}

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x3B
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Chunk section position
 | Long
 | Chunk section coordinate (encoded chunk x and z with each 22 bits, and section y with 20 bits, from left to right).
 |-
 | 
 | Boolean
 | Always inverse the preceding Update Light packet's "Trust Edges" bool
 |-
 | Blocks array size
 | VarInt
 | Number of elements in the following array.
 |-
 | Blocks
 | Array of VarLong
 | Each entry is composed of the block state id, shifted right by 12, and the relative block position in the chunk section (4 bits for x, z, and y, from left to right).
 |}

Chunk section position is encoded: 
<syntaxhighlight lang="java">
((sectionX & 0x3FFFFF) << 42) | (sectionY & 0xFFFFF) | ((sectionZ & 0x3FFFFF) << 20);
</syntaxhighlight>
and decoded:
<syntaxhighlight lang="java">
sectionX = long >> 42;
sectionY = long << 44 >> 44;
sectionZ = long << 22 >> 42;
</syntaxhighlight>

Blocks are encoded: 
<syntaxhighlight lang="java">
blockStateId << 12 | (blockLocalX << 8 | blockLocalZ << 4 | blockLocalY)
//Uses the local position of the given block position relative to its respective chunk section
</syntaxhighlight>

==== Select Advancement Tab ====

Sent by the server to indicate that the client should switch advancement tab. Sent either when the client switches tab in the GUI or when an advancement in another tab is made.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x3C
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Has id
 | Boolean
 | Indicates if the next field is present.
 |-
 | Optional Identifier
 | String (32767)
 | See below.
 |}

The Identifier can be one of the following:

{| class="wikitable"
 ! Optional Identifier
 |-
 | minecraft:story/root
 |-
 | minecraft:nether/root
 |-
 | minecraft:end/root
 |-
 | minecraft:adventure/root
 |-
 | minecraft:husbandry/root
 |}

If no or an invalid identifier is sent, the client will switch to the first tab in the GUI.

==== World Border ==== 

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="18"| 0x3D
 |rowspan="18"| Play
 |rowspan="18"| Client
 |colspan="2"| Action
 | VarInt Enum
 | Determines the format of the rest of the packet.
 |-
 ! Action
 ! Field Name
 ! 
 ! 
 |-
 | 0: set size
 | Diameter
 | Double
 | Length of a single side of the world border, in meters.
 |-
 |rowspan="3"| 1: lerp size
 | Old Diameter
 | Double
 | Current length of a single side of the world border, in meters.
 |-
 | New Diameter
 | Double
 | Target length of a single side of the world border, in meters.
 |-
 | Speed
 | VarLong
 | Number of real-time ''milli''seconds until New Diameter is reached. It appears that Notchian server does not sync world border speed to game ticks, so it gets out of sync with server lag. If the world border is not moving, this is set to 0.
 |-
 |rowspan="2"| 2: set center
 | X
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 |rowspan="8"| 3: initialize
 | X
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Old Diameter
 | Double
 | Current length of a single side of the world border, in meters.
 |-
 | New Diameter
 | Double
 | Target length of a single side of the world border, in meters.
 |-
 | Speed
 | VarLong
 | Number of real-time ''milli''seconds until New Diameter is reached. It appears that Notchian server does not sync world border speed to game ticks, so it gets out of sync with server lag. If the world border is not moving, this is set to 0.
 |-
 | Portal Teleport Boundary
 | VarInt
 | Resulting coordinates from a portal teleport are limited to ±value. Usually 29999984.
 |-
 | Warning Blocks
 | VarInt
 | In meters.
 |-
 | Warning Time
 | VarInt
 | In seconds as set by <code>/worldborder warning time</code>.
 |-
 | 4: set warning time
 | Warning Time
 | VarInt
 | In seconds as set by <code>/worldborder warning time</code>.
 |-
 | 5: set warning blocks
 | Warning Blocks
 | VarInt
 | In meters.
 |}

The Notchian client determines how solid to display the warning by comparing to whichever is higher, the warning distance or whichever is lower, the distance from the current diameter to the target diameter or the place the border will be after warningTime seconds. In pseudocode:

<syntaxhighlight lang="java">
distance = max(min(resizeSpeed * 1000 * warningTime, abs(targetDiameter - currentDiameter)), warningDistance);
if (playerDistance < distance) {
    warning = 1.0 - playerDistance / distance;
} else {
    warning = 0.0;
}
</syntaxhighlight>

==== Camera ====

Sets the entity that the player renders from. This is normally used when the player left-clicks an entity while in spectator mode.

The player's camera will move with the entity and look where it is looking. The entity is often another player, but can be any type of entity.  The player is unable to move this entity (move packets will act as if they are coming from the other entity).

If the given entity is not loaded by the player, this packet is ignored.  To return control to the player, send this packet with their entity ID.

The Notchian server resets this (sends it back to the default entity) whenever the spectated entity is killed or the player sneaks, but only if they were spectating an entity. It also sends this packet whenever the player switches out of spectator mode (even if they weren't spectating an entity).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x3E
 | Play
 | Client
 | Camera ID
 | VarInt
 | ID of the entity to set the client's camera to.
 |}

The notchian client also loads certain shaders for given entities:

* Creeper &rarr; <code>shaders/post/creeper.json</code>
* Spider (and cave spider) &rarr; <code>shaders/post/spider.json</code>
* Enderman &rarr; <code>shaders/post/invert.json</code>
* Anything else &rarr; the current shader is unloaded

==== Held Item Change (clientbound) ====

Sent to change the player's slot selection.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x3F
 | Play
 | Client
 | Slot
 | Byte
 | The slot which the player has selected (0–8).
 |}

==== Update View Position ====

{{Need Info|Why is this even needed?  Is there a better name for it?  My guess is that it's something to do with logical behavior with latency, but it still seems weird.}}

Updates the client's location.  This is used to determine what chunks should remain loaded and if a chunk load should be ignored; chunks outside of the view distance may be unloaded.

Sent whenever the player moves across a chunk border horizontally, and also (according to testing) for any integer change in the vertical axis, even if it doesn't go across a chunk section border.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x40
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Chunk X
 | VarInt
 | Chunk X coordinate of the player's position.
 |-
 | Chunk Z
 | VarInt
 | Chunk Z coordinate of the player's position.
 |}

==== Update View Distance ====

Sent by the integrated singleplayer server when changing render distance.  Does not appear to be used by the dedicated server, as <code>view-distance</code> in server.properties cannot be changed at runtime.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x41
 | Play
 | Client
 | View Distance
 | VarInt
 | Render distance (2-32).
 |}
 
==== Spawn Position ====

Sent by the server after login to specify the coordinates of the spawn point (the point at which players spawn at, and which the compass points to). It can be sent at any time to update the point compasses point at.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x42
 | Play
 | Client
 | Location
 | Position
 | Spawn location.
 |}

==== Display Scoreboard ====

This is sent to the client when it should display a scoreboard.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x43
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Position
 | Byte
 | The position of the scoreboard. 0: list, 1: sidebar, 2: below name, 3 - 18: team specific sidebar, indexed as 3 + team color.
 |-
 | Score Name
 | String (16)
 | The unique name for the scoreboard to be displayed.
 |}

==== Entity Metadata ====

Updates one or more [[Entities#Entity Metadata Format|metadata]] properties for an existing entity. Any properties not included in the Metadata field are left unchanged.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x44
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Metadata
 | [[Entities#Entity Metadata Format|Entity Metadata]]
 | 
 |}

==== Attach Entity ====

This packet is sent when an entity has been {{Minecraft Wiki|Lead|leashed}} to another entity.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x45
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Attached Entity ID
 | Int
 | Attached entity's EID.
 |-
 | Holding Entity ID
 | Int
 | ID of the entity holding the lead. Set to -1 to detach.
 |}

==== Entity Velocity ====

Velocity is believed to be in units of 1/8000 of a block per server tick (50ms); for example, -1343 would move (-1343 / 8000) = −0.167875 blocks per tick (or −3,3575 blocks per second).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x46
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Velocity X
 | Short
 | Velocity on the X axis.
 |-
 | Velocity Y
 | Short
 | Velocity on the Y axis.
 |-
 | Velocity Z
 | Short
 | Velocity on the Z axis.
 |}

==== Entity Equipment ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="3"| 0x47
 |rowspan="3"| Play
 |rowspan="3"| Client
 |colspan="2"| Entity ID
 |colspan="2"| VarInt
 | Entity's EID.
 |-
 |rowspan="2"| Equipment
 | Slot
 |rowspan="2"| Array
 | Byte Enum
 | Equipment slot. 0: main hand, 1: off hand, 2–5: armor slot (2: boots, 3: leggings, 4: chestplate, 5: helmet).  Also has the top bit set if another entry follows, and otherwise unset if this is the last item in the array.
 |-
 | Item
 | [[Slot Data|Slot]]
 |
 |}

==== Set Experience ====

Sent by the server when the client should change experience levels.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x48
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Experience bar
 | Float
 | Between 0 and 1.
 |-
 | Level
 | VarInt
 | 
 |-
 | Total Experience
 | VarInt
 | See {{Minecraft Wiki|Experience#Leveling up}} on the Minecraft Wiki for Total Experience to Level conversion.
 |}

==== Update Health ====

Sent by the server to update/set the health of the player it is sent to.

Food {{Minecraft Wiki|Food#Hunger vs. Saturation|saturation}} acts as a food “overcharge”. Food values will not decrease while the saturation is over zero. Players logging in automatically get a saturation of 5.0. Eating food increases the saturation as well as the food bar.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x49
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Health
 | Float
 | 0 or less = dead, 20 = full HP.
 |-
 | Food
 | VarInt
 | 0–20.
 |- 
 | Food Saturation
 | Float
 | Seems to vary from 0.0 to 5.0 in integer increments.
 |}

==== Scoreboard Objective ====

This is sent to the client when it should create a new {{Minecraft Wiki|scoreboard}} objective or remove one.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x4A
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Objective Name
 | String (16)
 | A unique name for the objective.
 |-
 | Mode
 | Byte
 | 0 to create the scoreboard. 1 to remove the scoreboard. 2 to update the display text. 
 |-
 | Objective Value
 | Optional Chat
 | Only if mode is 0 or 2. The text to be displayed for the score.
 |-
 | Type
 | Optional VarInt enum
 | Only if mode is 0 or 2. 0 = "integer", 1 = "hearts".
 |}

==== Set Passengers ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x4B
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Entity ID
 | VarInt
 | Vehicle's EID.
 |-
 | Passenger Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Passengers
 | Array of VarInt
 | EIDs of entity's passengers.
 |}

==== Teams ====

Creates and updates teams.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="23"| 0x4C
 |rowspan="23"| Play
 |rowspan="23"| Client
 |colspan="2"| Team Name
 | String (16)
 | A unique name for the team. (Shared with scoreboard).
 |-
 |colspan="2"| Mode
 | Byte
 | Determines the layout of the remaining packet.
 |-
 |rowspan="9"| 0: create team
 | Team Display Name
 | Chat
 | 
 |-
 | Friendly Flags
 | Byte
 | Bit mask. 0x01: Allow friendly fire, 0x02: can see invisible players on same team.
 |-
 | Name Tag Visibility
 | String Enum (32)
 | <code>always</code>, <code>hideForOtherTeams</code>, <code>hideForOwnTeam</code>, <code>never</code>.
 |-
 | Collision Rule
 | String Enum (32)
 | <code>always</code>, <code>pushOtherTeams</code>, <code>pushOwnTeam</code>, <code>never</code>.
 |-
 | Team Color
 | VarInt enum
 | Used to color the name of players on the team; see below.
 |-
 | Team Prefix
 | Chat
 | Displayed before the names of players that are part of this team.
 |-
 | Team Suffix
 | Chat
 | Displayed after the names of players that are part of this team.
 |-
 | Entity Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Entities
 | Array of String (40)
 | Identifiers for the entities in this team.  For players, this is their username; for other entities, it is their UUID.
 |-
 | 1: remove team
 | ''no fields''
 | ''no fields''
 | 
 |-
 |rowspan="7"| 2: update team info
 | Team Display Name
 | Chat
 | 
 |-
 | Friendly Flags
 | Byte
 | Bit mask. 0x01: Allow friendly fire, 0x02: can see invisible entities on same team.
 |-
 | Name Tag Visibility
 | String Enum (32)
 | <code>always</code>, <code>hideForOtherTeams</code>, <code>hideForOwnTeam</code>, <code>never</code>
 |-
 | Collision Rule
 | String Enum (32)
 | <code>always</code>, <code>pushOtherTeams</code>, <code>pushOwnTeam</code>, <code>never</code>
 |-
 | Team Color
 | VarInt enum
 | Used to color the name of players on the team; see below.
 |-
 | Team Prefix
 | Chat
 | Displayed before the names of players that are part of this team.
 |-
 | Team Suffix
 | Chat
 | Displayed after the names of players that are part of this team.
 |-
 |rowspan="2"| 3: add entities to team
 | Entity Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Entities
 | Array of String (40)
 | Identifiers for the added entities.  For players, this is their username; for other entities, it is their UUID.
 |-
 |rowspan="2"| 4: remove entities from team
 | Entity Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Entities
 | Array of String (40)
 | Identifiers for the removed entities.  For players, this is their username; for other entities, it is their UUID.
 |}

Team Color: The color of a team defines how the names of the team members are visualized; any formatting code can be used. The following table lists all the possible values.

{| class="wikitable"
 ! ID
 ! Formatting
 |-
 | 0-15
 | Color formatting, same values as [[Chat]] colors.
 |-
 | 16
 | Obfuscated
 |-
 | 17
 | Bold
 |-
 | 18
 | Strikethrough
 |-
 | 19
 | Underlined
 |-
 | 20
 | Italic
 |-
 | 21
 | Reset
 |}

==== Update Score ====

This is sent to the client when it should update a scoreboard item. 

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x4D
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Entity Name
 | String (40)
 | The entity whose score this is.  For players, this is their username; for other entities, it is their UUID.
 |-
 | Action
 | Byte
 | 0 to create/update an item. 1 to remove an item.
 |-
 | Objective Name
 | String (16)
 | The name of the objective the score belongs to.
 |-
 | Value
 | Optional VarInt
 | The score to be displayed next to the entry. Only sent when Action does not equal 1.
 |}

==== Time Update ====

Time is based on ticks, where 20 ticks happen every second. There are 24000 ticks in a day, making Minecraft days exactly 20 minutes long.

The time of day is based on the timestamp modulo 24000. 0 is sunrise, 6000 is noon, 12000 is sunset, and 18000 is midnight.

The default SMP server increments the time by <code>20</code> every second.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x4E
 |rowspan="2"| Play
 |rowspan="2"| Client
 | World Age
 | Long
 | In ticks; not changed by server commands.
 |-
 | Time of day
 | Long
 | The world (or region) time, in ticks. If negative the sun will stop moving at the Math.abs of the time.
 |}

==== Title ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="10"| 0x4F
 |rowspan="10"| Play
 |rowspan="10"| Client
 |colspan="2"| Action
 | VarInt Enum
 | 
 |-
 ! Action
 ! Field Name
 ! 
 ! 
 |-
 | 0: set title
 | Title Text
 | [[Chat]]
 | 
 |-
 | 1: set subtitle
 | Subtitle Text
 | [[Chat]]
 | 
 |- 
 | 2: set action bar
 | Action bar text
 | [[Chat]]
 | Displays a message above the hotbar (the same as position 2 in [[#Chat Message (clientbound)|Chat Message (clientbound)]].
 |-
 |rowspan="3"| 3: set times and display
 | Fade In
 | Int
 | Ticks to spend fading in.
 |-
 | Stay
 | Int
 | Ticks to keep the title displayed.
 |-
 | Fade Out
 | Int
 | Ticks to spend out, not when to start fading out.
 |-
 | 4: hide
 | ''no fields''
 | ''no fields''
 | 
 |-
 | 5: reset
 | ''no fields''
 | ''no fields''
 | 
 |}

“Hide” makes the title disappear, but if you run times again the same title will appear. “Reset” erases the text.

The title is visible on screen for Fade In + Stay + Fade Out ticks.

==== Entity Sound Effect ====

Plays a sound effect from an entity.

{{warning2|The pitch and volume fields in this packet are ignored.  See [https://bugs.mojang.com/browse/MC-138832 MC-138832] for more information.}}

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x50
 |rowspan="5"| Play
 |rowspan="5"| Client
 | Sound ID
 | VarInt
 | ID of hardcoded sound event ([https://pokechu22.github.io/Burger/1.16.5.html#sounds events] as of 1.16.5).
 |-
 | Sound Category
 | VarInt Enum
 | The category that this sound will be played from ([https://gist.github.com/konwboj/7c0c380d3923443e9d55 current categories]).
 |-
 | Entity ID
 | VarInt
 |
 |-
 | Volume
 | Float
 | 1.0 is 100%, capped between 0.0 and 1.0 by Notchian clients.
 |-
 | Pitch
 | Float
 | Float between 0.5 and 2.0 by Notchian clients.
 |}

==== Sound Effect ====

This packet is used to play a number of hardcoded sound events. For custom sounds, use [[#Named Sound Effect|Named Sound Effect]].

{{Warning|Numeric sound effect IDs are liable to change between versions}}

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x51
 |rowspan="7"| Play
 |rowspan="7"| Client
 | Sound ID
 | VarInt
 | ID of hardcoded sound event ([https://pokechu22.github.io/Burger/1.16.5.html#sounds events] as of 1.16.5).
 |-
 | Sound Category
 | VarInt Enum
 | The category that this sound will be played from ([https://gist.github.com/konwboj/7c0c380d3923443e9d55 current categories]).
 |-
 | Effect Position X
 | Int
 | Effect X multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Effect Position Y
 | Int
 | Effect Y multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Effect Position Z
 | Int
 | Effect Z multiplied by 8 ([[Data types#Fixed-point numbers|fixed-point number]] with only 3 bits dedicated to the fractional part).
 |-
 | Volume
 | Float
 | 1.0 is 100%, capped between 0.0 and 1.0 by Notchian clients.
 |-
 | Pitch
 | Float
 | Float between 0.5 and 2.0 by Notchian clients.
 |}

==== Stop Sound ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x52
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Flags
 | Byte
 | Controls which fields are present.
 |-
 | Source
 | Optional VarInt enum
 | Only if flags is 3 or 1 (bit mask 0x1).  See below.  If not present, then sounds from all sources are cleared.
 |-
 | Sound
 | Optional Identifier
 | Only if flags is 2 or 3 (bit mask 0x2).  A sound effect name, see [[#Named Sound Effect|Named Sound Effect]].  If not present, then all sounds are cleared.
 |}

Categories:

{| class="wikitable"
 ! Name !! Value
 |-
 | master || 0
 |-
 | music || 1
 |-
 | record || 2
 |-
 | weather || 3
 |-
 | block || 4
 |-
 | hostile || 5
 |-
 | neutral || 6
 |-
 | player || 7
 |-
 | ambient || 8
 |-
 | voice || 9
 |}

==== Player List Header And Footer ====

This packet may be used by custom servers to display additional information above/below the player list. It is never sent by the Notchian server.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x53
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Header
 | [[Chat]]
 | To remove the header, send a empty text component: <code>{"text":""}</code>.
 |-
 | Footer
 | [[Chat]]
 | To remove the footer, send a empty text component: <code>{"text":""}</code>.
 |}

==== NBT Query Response ====

Sent in response to [[#Query Block NBT|Query Block NBT]] or [[#Query Entity NBT|Query Entity NBT]].

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x54
 |rowspan="2"| Play
 |rowspan="2"| Client
 | Transaction ID
 | VarInt
 | Can be compared to the one sent in the original query packet.
 |-
 | NBT
 | [[NBT|NBT Tag]]
 | The NBT of the block or entity.  May be a TAG_END (0) in which case no NBT is present.
 |}

==== Collect Item ====

Sent by the server when someone picks up an item lying on the ground — its sole purpose appears to be the animation of the item flying towards you. It doesn't destroy the entity in the client memory, and it doesn't add it to your inventory. The server only checks for items to be picked up after each [[#Player Position|Player Position]] (and [[#Player Position And Look|Player Position And Look]]) packet sent by the client. The collector entity can be any entity; it does not have to be a player. The collected entity also can be any entity, but the Notchian server only uses this for items, experience orbs, and the different varieties of arrows.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x55
 |rowspan="3"| Play
 |rowspan="3"| Client
 | Collected Entity ID
 | VarInt
 | 
 |- 
 | Collector Entity ID
 | VarInt
 | 
 |- 
 | Pickup Item Count
 | VarInt
 | Seems to be 1 for XP orbs, otherwise the number of items in the stack.
 |}

==== Entity Teleport ====

This packet is sent by the server when an entity moves more than 8 blocks.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x56
 |rowspan="7"| Play
 |rowspan="7"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | X
 | Double
 | 
 |-
 | Y
 | Double
 | 
 |-
 | Z
 | Double
 | 
 |-
 | Yaw
 | Angle
 | New angle, not a delta.
 |-
 | Pitch
 | Angle
 | New angle, not a delta.
 |-
 | On Ground
 | Boolean
 | 
 |}

==== Advancements ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="9"| 0x57
 |rowspan="9"| Play
 |rowspan="9"| Client
 |colspan="2"| Reset/Clear
 |colspan="2"| Boolean
 | Whether to reset/clear the current advancements.
 |-
 |colspan="2"| Mapping size
 |colspan="2"| VarInt
 | Size of the following array.
 |-
 |rowspan="2"| Advancement mapping
 | Key
 |rowspan="2"| Array
 | Identifier
 | The identifier of the advancement.
 |-
 | Value
 | Advancement
 | See below
 |-
 |colspan="2"| List size
 |colspan="2"| VarInt
 | Size of the following array.
 |-
 |colspan="2"| Identifiers
 |colspan="2"| Array of Identifier
 | The identifiers of the advancements that should be removed.
 |-
 |colspan="2"| Progress size
 |colspan="2"| VarInt
 | Size of the following array.
 |-
 |rowspan="2"| Progress mapping
 | Key
 |rowspan="2"| Array
 | Identifier
 | The identifier of the advancement.
 |-
 | Value
 | Advancement progress
 | See below.
 |}

Advancement structure:

{| class="wikitable"
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |colspan="2"| Has parent
 |colspan="2"| Boolean
 | Indicates whether the next field exists.
 |-
 |colspan="2"| Parent id
 |colspan="2"| Optional Identifier
 | The identifier of the parent advancement.
 |-
 |colspan="2"| Has display
 |colspan="2"| Boolean
 | Indicates whether the next field exists.
 |-
 |colspan="2"| Display data
 |colspan="2"| Optional advancement display
 | See below.
 |-
 |colspan="2"| Number of criteria
 |colspan="2"| VarInt
 | Size of the following array.
 |-
 |rowspan="2"| Criteria
 | Key
 |rowspan="2"| Array
 | Identifier
 | The identifier of the criterion.
 |-
 | Value
 | '''Void'''
 | There is ''no'' content written here. Perhaps this will be expanded in the future?
 |-
 |colspan="2"| Array length
 |colspan="2"| VarInt
 | Number of arrays in the following array.
 |-
 |rowspan="2"| Requirements
 | Array length 2
 |rowspan="2"| Array
 | VarInt
 | Number of elements in the following array.
 |-
 | Requirement
 | Array of String
 | Array of required criteria.
 |}

Advancement display:

{| class="wikitable"
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | Title
 | Chat
 |
 |-
 | Description
 | Chat
 |
 |-
 | Icon
 | [[Slot]]
 |
 |-
 | Frame type
 | VarInt enum
 | 0 = <code>task</code>, 1 = <code>challenge</code>, 2 = <code>goal</code>.
 |-
 | Flags
 | Integer
 | 0x1: has background texture; 0x2: <code>show_toast</code>; 0x4: <code>hidden</code>.
 |-
 | Background texture
 | Optional Identifier
 | Background texture location.  Only if flags indicates it.
 |-
 | X coord
 | Float
 | 
 |-
 | Y coord
 | Float
 | 
 |}

Advancement progress:

{| class="wikitable"
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |colspan="2"| Size
 |colspan="2"| VarInt
 | Size of the following array.
 |-
 |rowspan="2"| Criteria
 | Criterion identifier
 |rowspan="2"| Array
 | Identifier
 | The identifier of the criterion.
 |-
 | Criterion progress
 | Criterion progress
 |
 |}

Criterion progress:

{| class="wikitable"
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | Achieved
 | Boolean
 | If true, next field is present.
 |-
 | Date of achieving
 | Optional Long
 | As returned by [https://docs.oracle.com/javase/6/docs/api/java/util/Date.html#getTime() <code>Date.getTime</code>].
 |}

==== Entity Properties ====

Sets {{Minecraft Wiki|Attribute|attributes}} on the given entity.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="6"| 0x58
 |rowspan="6"| Play
 |rowspan="6"| Client
 |colspan="2"| Entity ID
 |colspan="2"| VarInt
 | 
 |-
 |colspan="2"| Number Of Properties
 |colspan="2"| Int
 | Number of elements in the following array.
 |-
 |rowspan="4"| Property
 | Key
 |rowspan="4"| Array
 | Identifier
 | See below.
 |-
 | Value
 | Double
 | See below.
 |-
 | Number Of Modifiers
 | VarInt
 | Number of elements in the following array.
 |-
 | Modifiers
 | Array of Modifier Data
 | See {{Minecraft Wiki|Attribute#Modifiers}}. Modifier Data defined below.
 |}

Known Key values (see also {{Minecraft Wiki|Attribute#Modifiers}}):

{| class="wikitable"
 |-
 ! Key
 ! Default
 ! Min
 ! Max
 ! Label
 |-
 | generic.max_health
 | 20.0
 | 0.0
 | 1024.0
 | Max Health.
 |-
 | generic.follow_range
 | 32.0
 | 0.0
 | 2048.0
 | Follow Range.
 |-
 | generic.knockback_resistance
 | 0.0
 | 0.0
 | 1.0
 | Knockback Resistance.
 |-
 | generic.movement_speed
 | 0.7
 | 0.0
 | 1024.0
 | Movement Speed.
 |-
 | generic.attack_damage
 | 2.0
 | 0.0
 | 2048.0
 | Attack Damage.
 |-
 | generic.attack_speed
 | 4.0
 | 0.0
 | 1024.0
 | Attack Speed.
 |-
 | generic.flying_speed
 | 0.4
 | 0.0
 | 1024.0
 | Flying Speed.
 |-
 | generic.armor
 | 0.0
 | 0.0
 | 30.0
 | Armor.
 |-
 | generic.armor_toughness
 | 0.0
 | 0.0
 | 20.0
 | Armor Toughness.
 |-
 | generic.attack_knockback
 | 0.0
 | 0.0
 | 5.0
 | &mdash;
 |-
 | generic.luck
 | 0.0
 | -1024.0
 | 1024.0
 | Luck.
 |-
 | horse.jump_strength
 | 0.7
 | 0.0
 | 2.0
 | Jump Strength.
 |-
 | zombie.spawn_reinforcements
 | 0.0
 | 0.0
 | 1.0
 | Spawn Reinforcements Chance.
 |-
 | generic.reachDistance
 | 5.0
 | 0.0
 | 1024.0
 | Player Reach Distance (Forge only).
 |-
 | forge.swimSpeed
 | 1.0
 | 0.0
 | 1024.0
 | Swimming Speed (Forge only).
 |}

Unknown attributes will cause a game crash ([https://bugs.mojang.com/browse/MC-150405 MC-150405]) due to the default minimum being larger than the default value.

''Modifier Data'' structure:

{| class="wikitable"
 |-
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | UUID
 | UUID
 | 
 |-
 | Amount
 | Double
 | May be positive or negative.
 |-
 | Operation
 | Byte
 | See below.
 |}

The operation controls how the base value of the modifier is changed.

* 0: Add/subtract amount
* 1: Add/subtract amount percent of the current value
* 2: Multiply by amount percent

All of the 0's are applied first, and then the 1's, and then the 2's.

==== Entity Effect ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x59
 |rowspan="5"| Play
 |rowspan="5"| Client
 | Entity ID
 | VarInt
 | 
 |-
 | Effect ID
 | Byte
 | See {{Minecraft Wiki|Status effect#Effect IDs|this table}}.
 |-
 | Amplifier
 | Byte
 | Notchian client displays effect level as Amplifier + 1.
 |-
 | Duration
 | VarInt
 | Duration in ticks.
 |-
 | Flags
 | Byte
 | Bit field, see below.
 |}

Within flags:

* 0x01: Is ambient - was the effect spawned from a beacon?  All beacon-generated effects are ambient.  Ambient effects use a different icon in the HUD (blue border rather than gray).  If all effects on an entity are ambient, the [[Entities#Living Entity|"Is potion effect ambient" living metadata field]] should be set to true.  Usually should not be enabled.
* 0x02: Show particles - should all particles from this effect be hidden?  Effects with particles hidden are not included in the calculation of the effect color, and are not rendered on the HUD (but are still rendered within the inventory).  Usually should be enabled.
* 0x04: Show icon - should the icon be displayed on the client?  Usually should be enabled.

==== Declare Recipes ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |rowspan="4"| 0x5A
 |rowspan="4"| Play
 |rowspan="4"| Client
 |colspan="2"| Num Recipes
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="3"| Recipe
 | Type
 |rowspan="3"| Array
 | Identifier
 | The recipe type, see below.
 |-
 | Recipe ID
 | Identifier
 | 
 |-
 | Data
 | Optional, varies
 | Additional data for the recipe.  For some types, there will be no data.
 |}

Recipe types:

{| class="wikitable"
 ! Type
 ! Description
 ! Data
 |-
 | <code>crafting_shapeless</code>
 | Shapeless crafting recipe. All items in the ingredient list must be present, but in any order/slot.
 | As follows:
   {| class="wikitable"
    ! Name
    ! Type
    ! Description
    |-
    | Group
    | String
    | Used to group similar recipes together in the recipe book. Tag is present in recipe JSON.
    |-
    | Ingredient count
    | VarInt
    | Number of elements in the following array.
    |-
    | Ingredients
    | Array of Ingredient.
    |
    |-
    | Result
    | [[Slot]]
    |
    |}
 |-
 | <code>crafting_shaped</code>
 | Shaped crafting recipe. All items must be present in the same pattern (which may be flipped horizontally or translated).
 | As follows:
   {| class="wikitable"
    ! Name
    ! Type
    ! Description
    |-
    | Width
    | VarInt
    |
    |-
    | Height
    | VarInt
    |
    |-
    | Group
    | String
    | Used to group similar recipes together in the recipe book. Tag is present in recipe JSON.
    |-
    | Ingredients
    | Array of Ingredient
    | Length is <code>width * height</code>. Indexed by <code>x + (y * width)</code>.
    |-
    | Result
    | [[Slot]]
    |
    |}
 |-
 | <code>crafting_special_armordye</code>
 | Recipe for dying leather armor
 |rowspan="14"| None
 |-
 | <code>crafting_special_bookcloning</code>
 | Recipe for copying contents of written books
 |-
 | <code>crafting_special_mapcloning</code>
 | Recipe for copying maps
 |-
 | <code>crafting_special_mapextending</code>
 | Recipe for adding paper to maps
 |-
 | <code>crafting_special_firework_rocket</code>
 | Recipe for making firework rockets
 |-
 | <code>crafting_special_firework_star</code>
 | Recipe for making firework stars
 |-
 | <code>crafting_special_firework_star_fade</code>
 | Recipe for making firework stars fade between multiple colors
 |-
 | <code>crafting_special_repairitem</code>
 | Recipe for repairing items via crafting
 |-
 | <code>crafting_special_tippedarrow</code>
 | Recipe for crafting tipped arrows
 |-
 | <code>crafting_special_bannerduplicate</code>
 | Recipe for copying banner patterns
 |-
 | <code>crafting_special_banneraddpattern</code>
 | Recipe for adding patterns to banners
 |-
 | <code>crafting_special_shielddecoration</code>
 | Recipe for applying a banner's pattern to a shield
 |-
 | <code>crafting_special_shulkerboxcoloring</code>
 | Recipe for recoloring a shulker box
 |-
 | <code>crafting_special_suspiciousstew</code>
 |
 |-
 | <code>smelting</code>
 | Smelting recipe
 |rowspan="4"| As follows:
   {| class="wikitable"
    ! Name
    ! Type
    ! Description
    |-
    | Group
    | String
    | Used to group similar recipes together in the recipe book.
    |-
    | Ingredient
    | Ingredient
    |
    |-
    | Result
    | [[Slot]]
    |
    |-
    | Experience
    | Float
    |
    |-
    | Cooking time
    | VarInt
    |
    |}
 |-
 | <code>blasting</code>
 | Blast furnace recipe
 |-
 | <code>smoking</code>
 | Smoker recipe
 |-
 | <code>campfire_cooking</code>
 | Campfire recipe
 |-
 | <code>stonecutting</code>
 | Stonecutter recipe
 | As follows:
   {| class="wikitable"
    ! Name
    ! Type
    ! Description
    |-
    | Group
    | String
    | Used to group similar recipes together in the recipe book.  Tag is present in recipe JSON.
    |-
    | Ingredient
    | Ingredient
    |
    |-
    | Result
    | [[Slot]]
    |
    |}
 |-
 | <code>smithing</code>
 | Smithing table recipe
 | As follows:
   {| class="wikitable"
    ! Name
    ! Type
    ! Description
    |-
    | Base
    | Ingredient
    | First item.
    |-
    | Addition
    | Ingredient
    | Second item.
    |-
    | Result
    | [[Slot]]
    |
    |}
 |}

Ingredient is defined as:

{| class="wikitable"
 ! Name
 ! Type
 ! Description
 |-
 | Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Items
 | Array of [[Slot]]
 | Any item in this array may be used for the recipe.  The count of each item should be 1.
 |}

==== Tags ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x5B
 |rowspan="4"| Play
 |rowspan="4"| Client
 | Block Tags
 | (See below)
 | IDs are block IDs.
 |-
 | Item Tags
 | (See below)
 | IDs are item IDs.
 |-
 | Fluid Tags
 | (See below)
 | IDs are fluid IDs.
 |-
 | Entity Tags
 | (See below)
 | IDs are entity IDs.
 |}

Tags look like:

{| class="wikitable"
 !colspan="2"| Field Name
 !colspan="2"| Field Type
 ! Notes
 |-
 |colspan="2"| Length
 |colspan="2"| VarInt
 | Number of elements in the following array.
 |-
 |rowspan="3"| Tags
 | Tag name
 |rowspan="3"| Array
 | Identifier
 |
 |-
 | Count
 | VarInt
 | Number of elements in the following array.
 |-
 | Entries
 | Array of VarInt
 | Numeric ID of the block/item.
 |}

More information on tags is available at: https://minecraft.gamepedia.com/Tag

And a list of all tags is here: https://minecraft.gamepedia.com/Tag#List_of_tags

=== Serverbound ===

==== Teleport Confirm ====

Sent by client as confirmation of [[#Player Position And Look (clientbound)|Player Position And Look]].

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x00
 | Play
 | Server
 | Teleport ID
 | VarInt
 | The ID given by the [[#Player Position And Look (clientbound)|Player Position And Look]] packet.
 |}

==== Query Block NBT ====

Used when <kbd>Shift</kbd>+<kbd>F3</kbd>+<kbd>I</kbd> is pressed while looking at a block.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x01
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Transaction ID
 | VarInt
 | An incremental ID so that the client can verify that the response matches.
 |-
 | Location
 | Position
 | The location of the block to check.
 |}

==== Query Entity NBT ====

Used when <kbd>Shift</kbd>+<kbd>F3</kbd>+<kbd>I</kbd> is pressed while looking at an entity.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x0D
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Transaction ID
 | VarInt
 | An incremental ID so that the client can verify that the response matches.
 |-
 | Entity ID
 | VarInt
 | The ID of the entity to query.
 |}

==== Set Difficulty ====

Must have at least op level 2 to use.  Appears to only be used on singleplayer; the difficulty buttons are still disabled in multiplayer.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x02
 | Play
 | Server
 | New difficulty
 | Byte
 | 0: peaceful, 1: easy, 2: normal, 3: hard .
 |}

==== Chat Message (serverbound) ====

Used to send a chat message to the server.  The message may not be longer than 256 characters or else the server will kick the client.

If the message starts with a <code>/</code>, the server will attempt to interpret it as a command.  Otherwise, the server will broadcast the same chat message to all players on the server (including the player that sent the message), prepended with player's name.  Specifically, it will respond with a translate [[chat]] component, "<code>chat.type.text</code>" with the first parameter set to the display name of the player (including some chat component logic to support clicking the name to send a PM) and the second parameter set to the message.  See [[Chat#Processing chat|processing chat]] for more information.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x03
 | Play
 | Server
 | Message
 | String (256)
 | The client sends the raw input, not a [[Chat]] component.
 |}

==== Client Status ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x04
 | Play
 | Server
 | Action ID
 | VarInt Enum
 | See below
 |}

''Action ID'' values:

{| class="wikitable"
 |-
 ! Action ID
 ! Action
 ! Notes
 |-
 | 0
 | Perform respawn
 | Sent when the client is ready to complete login and when the client is ready to respawn after death.
 |-
 | 1
 | Request stats
 | Sent when the client opens the Statistics menu.
 |}

==== Client Settings ====

Sent when the player connects, or when settings are changed.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="6"| 0x05
 |rowspan="6"| Play
 |rowspan="6"| Server
 | Locale
 | String (16)
 | e.g. <code>en_GB</code>.
 |-
 | View Distance
 | Byte
 | Client-side render distance, in chunks.
 |-
 | Chat Mode
 | VarInt Enum
 | 0: enabled, 1: commands only, 2: hidden.  See [[Chat#Processing chat|processing chat]] for more information.
 |-
 | Chat Colors
 | Boolean
 | “Colors” multiplayer setting.
 |-
 | Displayed Skin Parts
 | Unsigned Byte
 | Bit mask, see below.
 |-
 | Main Hand
 | VarInt Enum
 | 0: Left, 1: Right.
 |}

''Displayed Skin Parts'' flags:

* Bit 0 (0x01): Cape enabled
* Bit 1 (0x02): Jacket enabled
* Bit 2 (0x04): Left Sleeve enabled
* Bit 3 (0x08): Right Sleeve enabled
* Bit 4 (0x10): Left Pants Leg enabled
* Bit 5 (0x20): Right Pants Leg enabled
* Bit 6 (0x40): Hat enabled

The most significant bit (bit 7, 0x80) appears to be unused.

==== Tab-Complete (serverbound) ====

Sent when the client needs to tab-complete a <code>minecraft:ask_server</code> suggestion type.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x06
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Transaction Id
 | VarInt
 | The id of the transaction that the server will send back to the client in the response of this packet. Client generates this and increments it each time it sends another tab completion that doesn't get a response.
 |-
 | Text
 | String (32500)
 | All text behind the cursor without the <code>/</code> (e.g. to the left of the cursor in left-to-right languages like English).
 |}

==== Window Confirmation (serverbound) ====

If a confirmation sent by the client was not accepted, the server will reply with a [[#Window Confirmation (clientbound)|Window Confirmation (clientbound)]] packet with the Accepted field set to false. When this happens, the client must send this packet to apologize (as with movement), otherwise the server ignores any successive confirmations.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x07
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Window ID
 | Byte
 | The ID of the window that the action occurred in.
 |-
 | Action Number
 | Short
 | Every action that is to be accepted has a unique number. This number is an incrementing integer (starting at 1) with separate counts for each window ID.
 |-
 | Accepted
 | Boolean
 | Whether the action was accepted.
 |}

==== Click Window Button ====

Used when clicking on window buttons.  Until 1.14, this was only used by enchantment tables.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x08
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Window ID
 | Byte
 | The ID of the window sent by [[#Open Window|Open Window]].
 |-
 | Button ID
 | Byte
 | Meaning depends on window type; see below.
 |}

{| class="wikitable"
 ! Window type
 ! ID
 ! Meaning
 |-
 |rowspan="3"| Enchantment Table
 | 0 || Topmost enchantment.
 |-
 | 1 || Middle enchantment.
 |-
 | 2 || Bottom enchantment.
 |-
 |rowspan="4"| Lectern
 | 1 || Previous page (which does give a redstone output).
 |-
 | 2 || Next page.
 |-
 | 3 || Take Book.
 |-
 | 100+page || Opened page number - 100 + number.
 |-
 | Stonecutter
 |colspan="2"| Recipe button number - 4*row + col.  Depends on the item.
 |-
 | Loom
 |colspan="2"| Recipe button number - 4*row + col.  Depends on the item.
 |}

==== Click Window ====

This packet is sent by the client when the player clicks on a slot in a window.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="6"| 0x09
 |rowspan="6"| Play
 |rowspan="6"| Server
 | Window ID
 | Unsigned Byte
 | The ID of the window which was clicked. 0 for player inventory.
 |-
 | Slot
 | Short
 | The clicked slot number, see below.
 |-
 | Button
 | Byte
 | The button used in the click, see below.
 |-
 | Action Number
 | Short
 | A unique number for the action, implemented by Notchian as a counter, starting at 1 (different counter for every window ID). Used by the server to send back a [[#Window Confirmation (clientbound)|Window Confirmation (clientbound)]].
 |-
 | Mode
 | VarInt Enum
 | Inventory operation mode, see below.
 |-
 | Clicked item
 | [[Slot Data|Slot]]
 | The clicked slot. Has to be empty (item ID = -1) for drop mode. Is always empty for mode 2 and mode 5 packets. 
 |}

See [[Inventory]] for further information about how slots are indexed.

When right-clicking on a stack of items, half the stack will be picked up and half left in the slot. If the stack is an odd number, the half left in the slot will be smaller of the amounts.

The distinct type of click performed by the client is determined by the combination of the Mode and Button fields.

{| class="wikitable"
 ! Mode
 ! Button
 ! Slot
 ! Trigger
 |-
 !rowspan="4"| 0
 | 0
 | Normal
 | Left mouse click
 |-
 | 1
 | Normal
 | Right mouse click
 |-
 | 0
 | -999
 | Left click outside inventory (drop cursor stack)
 |-
 | 1
 | -999
 | Right click outside inventory (drop cursor single item)
 |-
 !rowspan="2"| 1
 | 0
 | Normal
 | Shift + left mouse click
 |-
 | 1
 | Normal
 | Shift + right mouse click ''(identical behavior)''
 |-
 !rowspan="5"| 2
 | 0
 | Normal
 | Number key 1
 |-
 | 1
 | Normal
 | Number key 2
 |-
 | 2
 | Normal
 | Number key 3
 |-
 | ⋮
 | ⋮
 | ⋮
 |-
 | 8
 | Normal
 | Number key 9
 |-
 ! 3
 | 2
 | Normal
 | Middle click, only defined for creative players in non-player inventories.
 |-
 !rowspan="1"| 4
 | 0
 | Normal*
 | Drop key (Q) (* Clicked item is always empty)
 |-
 !rowspan="9"| 5
 | 0
 | -999
 | Starting left mouse drag
 |-
 | 4
 | -999
 | Starting right mouse drag
 |-
 | 8
 | -999
 | Starting middle mouse drag, only defined for creative players in non-player inventories.  (Note: the vanilla client will still incorrectly send this for non-creative players - see [https://bugs.mojang.com/browse/MC-46584 MC-46584])
 |-
 | 1
 | Normal
 | Add slot for left-mouse drag
 |-
 | 5
 | Normal
 | Add slot for right-mouse drag
 |-
 | 9
 | Normal
 | Add slot for middle-mouse drag, only defined for creative players in non-player inventories.  (Note: the vanilla client will still incorrectly send this for non-creative players - see [https://bugs.mojang.com/browse/MC-46584 MC-46584])
 |-
 | 2
 | -999
 | Ending left mouse drag
 |-
 | 6
 | -999
 | Ending right mouse drag
 |-
 | 10
 | -999
 | Ending middle mouse drag, only defined for creative players in non-player inventories.  (Note: the vanilla client will still incorrectly send this for non-creative players - see [https://bugs.mojang.com/browse/MC-46584 MC-46584])
 |-
 ! 6
 | 0
 | Normal
 | Double click
 |}

Starting from version 1.5, “painting mode” is available for use in inventory windows. It is done by picking up stack of something (more than 1 item), then holding mouse button (left, right or middle) and dragging held stack over empty (or same type in case of right button) slots. In that case client sends the following to server after mouse button release (omitting first pickup packet which is sent as usual):

# packet with mode 5, slot -999, button (0 for left | 4 for right);
# packet for every slot painted on, mode is still 5, button (1 | 5);
# packet with mode 5, slot -999, button (2 | 6);

If any of the painting packets other than the “progress” ones are sent out of order (for example, a start, some slots, then another start; or a left-click in the middle) the painting status will be reset.

The server will send back a [[#Window Confirmation (clientbound)|Window Confirmation]] packet. If the click was not accepted, the client must send a matching [[#Window Confirmation (serverbound)|serverbound window confirmation]] packet before sending more [[#Click Window|Click Window]] packets, otherwise the server will reject them silently. The Notchian server also sends a [[#Window Items|Window Items]] packet for the open window and [[#Set Slot|Set Slot]] packets for the clicked and cursor slot, but only when the click was not accepted, probably to resynchronize client and server.

==== Close Window (serverbound) ====

This packet is sent by the client when closing a window.

Notchian clients send a Close Window packet with Window ID 0 to close their inventory even though there is never an [[#Open Window|Open Window]] packet for the inventory.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x0A
 | Play
 | Server
 | Window ID
 | Unsigned Byte
 | This is the ID of the window that was closed. 0 for player inventory.
 |}

==== Plugin Message (serverbound) ====

{{Main|Plugin channels}}

Mods and plugins can use this to send their data. Minecraft itself uses some [[plugin channel]]s. These internal channels are in the <code>minecraft</code> namespace.

More documentation on this: [http://dinnerbone.com/blog/2012/01/13/minecraft-plugin-channels-messaging/ http://dinnerbone.com/blog/2012/01/13/minecraft-plugin-channels-messaging/]

Note that the length of Data is known only from the packet length, since the packet has no length field of any kind.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x0B
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Channel
 | Identifier
 | Name of the [[plugin channel]] used to send the data.
 |-
 | Data
 | Byte Array
 | Any data, depending on the channel. <code>minecraft:</code> channels are documented [[plugin channel|here]]. The length of this array must be inferred from the packet length.
 |}

==== Edit Book ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x0C
 |rowspan="3"| Play
 |rowspan="3"| Server
 | New book
 | [[Slot]]
 |
 |-
 | Is signing
 | Boolean
 | True if the player is signing the book; false if the player is saving a draft.
 |-
 | Hand
 | VarInt enum
 | 0: Main hand, 1: Off hand.
 |}

When editing a draft, the [[NBT]] section of the Slot contains this:

<pre>
TAG_Compound(<nowiki>''</nowiki>): 1 entry
{
  TAG_List('pages'): 2 entries
  {
    TAG_String(0): 'Something on Page 1'
    TAG_String(1): 'Something on Page 2'
  }
}
</pre>

When signing the book, it instead looks like this:

<pre>
TAG_Compound(<nowiki>''</nowiki>): 3 entires
{
  TAG_String('author'): 'Steve'
  TAG_String('title'): 'A Wonderful Book'
  TAG_List('pages'): 2 entries
  {
    TAG_String(0): 'Something on Page 1'
    TAG_String(1): 'Something on Page 2'
  }
}
</pre>

==== Interact Entity ====

This packet is sent from the client to the server when the client attacks or right-clicks another entity (a player, minecart, etc).

A Notchian server only accepts this packet if the entity being attacked/used is visible without obstruction and within a 4-unit radius of the player's position.

The target X, Y, and Z fields represent the difference between the vector location of the cursor at the time of the packet and the entity's position.

Note that middle-click in creative mode is interpreted by the client and sent as a [[#Creative Inventory Action|Creative Inventory Action]] packet instead.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x0E
 |rowspan="7"| Play
 |rowspan="7"| Server
 | Entity ID
 | VarInt
 | The ID of the entity to interact.
 |-
 | Type
 | VarInt Enum
 | 0: interact, 1: attack, 2: interact at.
 |-
 | Target X
 | Optional Float
 | Only if Type is interact at.
 |-
 | Target Y
 | Optional Float
 | Only if Type is interact at.
 |-
 | Target Z
 | Optional Float
 | Only if Type is interact at.
 |-
 | Hand
 | Optional VarInt Enum
 | Only if Type is interact or interact at; 0: main hand, 1: off hand.
 |-
 | Sneaking
 | Boolean
 | If the client is sneaking.
 |}

==== Generate Structure ====

Sent when Generate is pressed on the {{Minecraft Wiki|Jigsaw Block}} interface.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x0F
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Location
 | Position
 | Block entity location.
 |-
 | Levels
 | VarInt
 | Value of the levels slider/max depth to generate.
 |-
 | Keep Jigsaws
 | Boolean
 |
 |}

==== Keep Alive (serverbound) ====

The server will frequently send out a keep-alive, each containing a random ID. The client must respond with the same packet.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x10
 | Play
 | Server
 | Keep Alive ID
 | Long
 | 
 |}

==== Lock Difficulty ====

Must have at least op level 2 to use.  Appears to only be used on singleplayer; the difficulty buttons are still disabled in multiplayer.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x11
 | Play
 | Server
 | Locked
 | Boolean
 |
 |}

==== Player Position ====

Updates the player's XYZ position on the server.

Checking for moving too fast is achieved like this:

* Each server tick, the player's current position is stored
* When a player moves, the changes in x, y, and z coordinates are compared with the positions from the previous tick (&Delta;x, &Delta;y, &Delta;z)
* Total movement distance squared is computed as &Delta;x&sup2; + &Delta;y&sup2; + &Delta;z&sup2;
* The expected movement distance squared is computed as velocityX&sup2; + veloctyY&sup2; + velocityZ&sup2;
* If the total movement distance squared value minus the expected movement distance squared value is more than 100 (300 if the player is using an elytra), they are moving too fast.

If the player is moving too fast, it will be logged that "<player> moved too quickly! " followed by the change in x, y, and z, and the player will be teleported back to their current (before this packet) serverside position.

Also, if the absolute value of X or the absolute value of Z is a value greater than 3.2×10<sup>7</sup>, or X, Y, or Z are not finite (either positive infinity, negative infinity, or NaN), the client will be kicked for “Invalid move player packet received”.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="4"| 0x12
 |rowspan="4"| Play
 |rowspan="4"| Server
 | X
 | Double
 | Absolute position.
 |-
 | Feet Y
 | Double
 | Absolute feet position, normally Head Y - 1.62.
 |-
 | Z
 | Double
 | Absolute position.
 |-
 | On Ground
 | Boolean
 | True if the client is on the ground, false otherwise.
 |}

==== Player Position And Rotation (serverbound) ====

A combination of [[#Player Rotation|Player Rotation]] and [[#Player Position|Player Position]].

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="6"| 0x13
 |rowspan="6"| Play
 |rowspan="6"| Server
 | X
 | Double
 | Absolute position.
 |-
 | Feet Y
 | Double
 | Absolute feet position, normally Head Y - 1.62.
 |-
 | Z
 | Double
 | Absolute position.
 |-
 | Yaw
 | Float
 | Absolute rotation on the X Axis, in degrees.
 |-
 | Pitch
 | Float
 | Absolute rotation on the Y Axis, in degrees.
 |-
 | On Ground
 | Boolean
 | True if the client is on the ground, false otherwise.
 |}

==== Player Rotation ====
[[File:Minecraft-trig-yaw.png|thumb|The unit circle for yaw]]
[[File:Yaw.png|thumb|The unit circle of yaw, redrawn]]

Updates the direction the player is looking in.

Yaw is measured in degrees, and does not follow classical trigonometry rules. The unit circle of yaw on the XZ-plane starts at (0, 1) and turns counterclockwise, with 90 at (-1, 0), 180 at (0,-1) and 270 at (1, 0). Additionally, yaw is not clamped to between 0 and 360 degrees; any number is valid, including negative numbers and numbers greater than 360.

Pitch is measured in degrees, where 0 is looking straight ahead, -90 is looking straight up, and 90 is looking straight down.

The yaw and pitch of player (in degrees), standing at point (x0, y0, z0) and looking towards point (x, y, z) can be calculated with:

 dx = x-x0
 dy = y-y0
 dz = z-z0
 r = sqrt( dx*dx + dy*dy + dz*dz )
 yaw = -atan2(dx,dz)/PI*180
 if yaw < 0 then
     yaw = 360 + yaw
 pitch = -arcsin(dy/r)/PI*180

You can get a unit vector from a given yaw/pitch via:

 x = -cos(pitch) * sin(yaw)
 y = -sin(pitch)
 z =  cos(pitch) * cos(yaw)

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x14
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Yaw
 | Float
 | Absolute rotation on the X Axis, in degrees.
 |-
 | Pitch
 | Float
 | Absolute rotation on the Y Axis, in degrees.
 |-
 | On Ground
 | Boolean
 | True if the client is on the ground, false otherwise.
 |}

==== Player Movement ====

This packet as well as [[#Player Position|Player Position]], [[#Player Look|Player Look]], and [[#Player Position And Look 2|Player Position And Look]] are called the “serverbound movement packets”. Vanilla clients will send Player Position once every 20 ticks even for a stationary player.

This packet is used to indicate whether the player is on ground (walking/swimming), or airborne (jumping/falling).

When dropping from sufficient height, fall damage is applied when this state goes from false to true. The amount of damage applied is based on the point where it last changed from true to false. Note that there are several movement related packets containing this state.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x15
 | Play
 | Server
 | On Ground
 | Boolean
 | True if the client is on the ground, false otherwise.
 |}

==== Vehicle Move (serverbound) ====

Sent when a player moves in a vehicle. Fields are the same as in [[#Player Position And Look (serverbound)|Player Position And Look]]. Note that all fields use absolute positioning and do not allow for relative positioning.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x16
 |rowspan="5"| Play
 |rowspan="5"| Server
 | X
 | Double
 | Absolute position (X coordinate).
 |-
 | Y
 | Double
 | Absolute position (Y coordinate).
 |-
 | Z
 | Double
 | Absolute position (Z coordinate).
 |-
 | Yaw
 | Float
 | Absolute rotation on the vertical axis, in degrees.
 |-
 | Pitch
 | Float
 | Absolute rotation on the horizontal axis, in degrees.
 |}

==== Steer Boat ====

Used to ''visually'' update whether boat paddles are turning.  The server will update the [[Entities#Boat|Boat entity metadata]] to match the values here.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x17
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Left paddle turning
 | Boolean
 |
 |-
 | Right paddle turning
 | Boolean
 |
 |}

Right paddle turning is set to true when the left button or forward button is held, left paddle turning is set to true when the right button or forward button is held.

==== Pick Item ====

Used to swap out an empty space on the hotbar with the item in the given inventory slot.  The Notchain client uses this for pick block functionality (middle click) to retrieve items from the inventory.


{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x18
 | Play
 | Server
 | Slot to use
 | VarInt
 | See [[Inventory]].
 |}

The server will first search the player's hotbar for an empty slot, starting from the current slot and looping around to the slot before it.  If there are no empty slots, it will start a second search from the current slot and find the first slot that does not contain an enchanted item.  If there still are no slots that meet that criteria, then the server will use the currently selected slot.

After finding the appropriate slot, the server swaps the items and then send 3 packets:

* [[Protocol#Set slot|Set Slot]], with window ID set to -2 and slot set to the newly chosen slot and the item set to the item that is now in that slot (which was previously at the slot the client requested)
* Set Slot, with window ID set to -2 and slot set to the slot the player requested, with the item that is now in that slot and was previously on the hotbar slot
* [[Protocol#Held_Item_Change_.28clientbound.29|Held Item Change]], with the slot set to the newly chosen slot.

==== Craft Recipe Request ====

This packet is sent when a player clicks a recipe in the crafting book that is craftable (white border).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x19
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Window ID
 | Byte
 |
 |-
 | Recipe
 | Identifier
 | A recipe ID.
 |-
 | Make all
 | Boolean
 | Affects the amount of items processed; true if shift is down when clicked.
 |}

==== Player Abilities (serverbound) ====

The vanilla client sends this packet when the player starts/stops flying with the Flags parameter changed accordingly.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x1A
 | Play
 | Server
 | Flags
 | Byte
 | Bit mask. 0x02: is flying.
 |}

==== Player Digging ====

Sent when the player mines a block. A Notchian server only accepts digging packets with coordinates within a 6-unit radius between the center of the block and 1.5 units from the player's feet (''not'' their eyes).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x1B
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Status
 | VarInt Enum
 | The action the player is taking against the block (see below).
 |-
 | Location
 | Position
 | Block position.
 |-
 | Face
 | Byte Enum
 | The face being hit (see below).
 |}

Status can be one of seven values:

{| class="wikitable"
 ! Value
 ! Meaning
 ! Notes
 |-
 | 0
 | Started digging
 | 
 |-
 | 1
 | Cancelled digging
 | Sent when the player lets go of the Mine Block key (default: left click).
 |-
 | 2
 | Finished digging
 | Sent when the client thinks it is finished.
 |-
 | 3
 | Drop item stack
 | Triggered by using the Drop Item key (default: Q) with the modifier to drop the entire selected stack (default: depends on OS). Location is always set to 0/0/0, Face is always set to -Y.
 |-
 | 4
 | Drop item
 | Triggered by using the Drop Item key (default: Q). Location is always set to 0/0/0, Face is always set to -Y.
 |-
 | 5
 | Shoot arrow / finish eating
 | Indicates that the currently held item should have its state updated such as eating food, pulling back bows, using buckets, etc. Location is always set to 0/0/0, Face is always set to -Y.
 |-
 | 6
 | Swap item in hand
 | Used to swap or assign an item to the second hand. Location is always set to 0/0/0, Face is always set to -Y.
 |}

The Face field can be one of the following values, representing the face being hit:

{| class="wikitable"
 |-
 ! Value
 ! Offset
 ! Face
 |-
 | 0
 | -Y
 | Bottom
 |-
 | 1
 | +Y
 | Top
 |-
 | 2
 | -Z
 | North
 |-
 | 3
 | +Z
 | South
 |-
 | 4
 | -X
 | West
 |-
 | 5
 | +X
 | East
 |}

==== Entity Action ====

Sent by the client to indicate that it has performed certain actions: sneaking (crouching), sprinting, exiting a bed, jumping with a horse, and opening a horse's inventory while riding it.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x1C
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Entity ID
 | VarInt
 | Player ID
 |-
 | Action ID
 | VarInt Enum
 | The ID of the action, see below.
 |-
 | Jump Boost
 | VarInt
 | Only used by the “start jump with horse” action, in which case it ranges from 0 to 100. In all other cases it is 0.
 |}

Action ID can be one of the following values:

{| class="wikitable"
 ! ID
 ! Action
 |-
 | 0
 | Start sneaking
 |-
 | 1
 | Stop sneaking
 |-
 | 2
 | Leave bed
 |-
 | 3
 | Start sprinting
 |-
 | 4
 | Stop sprinting
 |-
 | 5
 | Start jump with horse
 |-
 | 6
 | Stop jump with horse
 |-
 | 7
 | Open horse inventory
 |-
 | 8
 | Start flying with elytra
 |}

Leave bed is only sent when the “Leave Bed” button is clicked on the sleep GUI, not when waking up due today time.

Open horse inventory is only sent when pressing the inventory key (default: E) while on a horse — all other methods of opening a horse's inventory (involving right-clicking or shift-right-clicking it) do not use this packet.

==== Steer Vehicle ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x1D
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Sideways
 | Float
 | Positive to the left of the player.
 |-
 | Forward
 | Float
 | Positive forward.
 |-
 | Flags
 | Unsigned Byte
 | Bit mask. 0x1: jump, 0x2: unmount.
 |}

Also known as 'Input' packet.

==== Set Recipe Book State ====

Replaces Recipe Book Data, type 1.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x1E
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Book ID
 | VarInt enum
 | 0: crafting, 1: furnace, 2: blast furnace, 3: smoker.
 |-
 | Book Open
 | Boolean
 |
 |-
 | Filter Active
 | Boolean
 |
 |}

==== Set Displayed Recipe ====

Replaces Recipe Book Data, type 0.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x1F
 | Play
 | Server
 | Recipe ID
 | Identifier
 |
 |}

==== Name Item ====

Sent as a player is renaming an item in an anvil (each keypress in the anvil UI sends a new Name Item packet).  If the new name is empty, then the item loses its custom name (this is different from setting the custom name to the normal name of the item).  The item name may be no longer than 35 characters long, and if it is longer than that, then the rename is silently ignored.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x20
 | Play
 | Server
 | Item name
 | String (32767)
 | The new name of the item.
 |}

==== Resource Pack Status ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x21
 | Play
 | Server
 | Result
 | VarInt Enum
 | 0: successfully loaded, 1: declined, 2: failed download, 3: accepted.
 |}

==== Advancement Tab ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x22
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Action
 | VarInt enum
 | 0: Opened tab, 1: Closed screen.
 |-
 | Tab ID
 | Optional identifier
 | Only present if action is Opened tab.
 |}

==== Select Trade ====

When a player selects a specific trade offered by a villager NPC.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x23
 | Play
 | Server
 | Selected slot
 | VarInt
 | The selected slot in the players current (trading) inventory. (Was a full Integer for the plugin message).
 |}

==== Set Beacon Effect ====

Changes the effect of the current beacon.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x24
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Primary Effect
 | VarInt
 | A [http://minecraft.gamepedia.com/Data_values#Potions Potion ID]. (Was a full Integer for the plugin message).
 |-
 | Secondary Effect
 | VarInt
 | A [http://minecraft.gamepedia.com/Data_values#Potions Potion ID]. (Was a full Integer for the plugin message).
 |}

==== Held Item Change (serverbound) ====

Sent when the player changes the slot selection

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x25
 | Play
 | Server
 | Slot
 | Short
 | The slot which the player has selected (0–8).
 |}

==== Update Command Block ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x26
 |rowspan="5"| Play
 |rowspan="5"| Server
 |-
 | Location
 | Position
 |
 |-
 | Command
 | String (32767)
 |
 |-
 | Mode || VarInt enum || One of SEQUENCE (0), AUTO (1), or REDSTONE (2).
 |-
 | Flags
 | Byte
 | 0x01: Track Output (if false, the output of the previous command will not be stored within the command block); 0x02: Is conditional; 0x04: Automatic.
 |}

==== Update Command Block Minecart ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="3"| 0x27
 |rowspan="3"| Play
 |rowspan="3"| Server
 | Entity ID
 | VarInt
 |
 |-
 | Command
 | String
 |
 |-
 | Track Output
 | Boolean
 | If false, the output of the previous command will not be stored within the command block.
 |}

==== Creative Inventory Action ====

While the user is in the standard inventory (i.e., not a crafting bench) in Creative mode, the player will send this packet.

Clicking in the creative inventory menu is quite different from non-creative inventory management. Picking up an item with the mouse actually deletes the item from the server, and placing an item into a slot or dropping it out of the inventory actually tells the server to create the item from scratch. (This can be verified by clicking an item that you don't mind deleting, then severing the connection to the server; the item will be nowhere to be found when you log back in.) As a result of this implementation strategy, the "Destroy Item" slot is just a client-side implementation detail that means "I don't intend to recreate this item.". Additionally, the long listings of items (by category, etc.) are a client-side interface for choosing which item to create. Picking up an item from such listings sends no packets to the server; only when you put it somewhere does it tell the server to create the item in that location.

This action can be described as "set inventory slot". Picking up an item sets the slot to item ID -1. Placing an item into an inventory slot sets the slot to the specified item. Dropping an item (by clicking outside the window) effectively sets slot -1 to the specified item, which causes the server to spawn the item entity, etc.. All other inventory slots are numbered the same as the non-creative inventory (including slots for the 2x2 crafting menu, even though they aren't visible in the vanilla client).

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="2"| 0x28
 |rowspan="2"| Play
 |rowspan="2"| Server
 | Slot
 | Short
 | Inventory slot.
 |-
 | Clicked Item
 | [[Slot Data|Slot]]
 | 
 |}

==== Update Jigsaw Block ====

Sent when Done is pressed on the {{Minecraft Wiki|Jigsaw Block}} interface.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="6"| 0x29
 |rowspan="6"| Play
 |rowspan="6"| Server
 | Location
 | Position
 | Block entity location
 |-
 | Name
 | Identifier
 | 
 |-
 | Target
 | Identifier
 | 
 |- 
 | Pool
 | Identifier
 | 
 |-
 | Final state
 | String
 | "Turns into" on the GUI, <code>final_state</code> in NBT.
 |-
 | Joint type
 | String
 | <code>rollable</code> if the attached piece can be rotated, else <code>aligned</code>.
 |}

==== Update Structure Block ====

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="17"| 0x2A
 |rowspan="17"| Play
 |rowspan="17"| Server
 |-
 | Location
 | Position
 | Block entity location.
 |-
 | Action
 | VarInt enum
 | An additional action to perform beyond simply saving the given data; see below.
 |-
 | Mode
 | VarInt enum
 | One of SAVE (0), LOAD (1), CORNER (2), DATA (3).
 |-
 | Name
 | String
 |
 |-
 | Offset X || Byte
 | Between -32 and 32.
 |-
 | Offset Y || Byte
 | Between -32 and 32.
 |-
 | Offset Z || Byte
 | Between -32 and 32.
 |-
 | Size X || Byte
 | Between 0 and 32.
 |-
 | Size Y || Byte
 | Between 0 and 32.
 |-
 | Size Z || Byte
 | Between 0 and 32.
 |-
 | Mirror
 | VarInt enum
 | One of NONE (0), LEFT_RIGHT (1), FRONT_BACK (2).
 |-
 | Rotation
 | VarInt enum
 | One of NONE (0), CLOCKWISE_90 (1), CLOCKWISE_180 (2), COUNTERCLOCKWISE_90 (3).
 |-
 | Metadata
 | String
 | 
 |-
 | Integrity
 | Float
 | Between 0 and 1.
 |-
 |Seed
 |VarLong
 |
 |-
 | Flags
 | Byte
 | 0x01: Ignore entities; 0x02: Show air; 0x04: Show bounding box.
 |}

Possible actions:

* 0 - Update data
* 1 - Save the structure
* 2 - Load the structure
* 3 - Detect size

The Notchian client uses update data to indicate no special action should be taken (i.e. the done button).

==== Update Sign ====

{{anchor|Update Sign (serverbound)}}This message is sent from the client to the server when the “Done” button is pushed after placing a sign.

The server only accepts this packet after [[#Open Sign Editor|Open Sign Editor]], otherwise this packet is silently ignored.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="5"| 0x2B
 |rowspan="5"| Play
 |rowspan="5"| Server
 | Location
 | Position
 | Block Coordinates.
 |-
 | Line 1
 | String (384)
 | First line of text in the sign.
 |-
 | Line 2
 | String (384)
 | Second line of text in the sign.
 |-
 | Line 3
 | String (384)
 | Third line of text in the sign.
 |-
 | Line 4
 | String (384)
 | Fourth line of text in the sign.
 |}

==== Animation (serverbound) ====

Sent when the player's arm swings.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2C
 | Play
 | Server
 | Hand
 | VarInt Enum
 | Hand used for the animation. 0: main hand, 1: off hand.
 |}

==== Spectate ====

Teleports the player to the given entity.  The player must be in spectator mode.

The Notchian client only uses this to teleport to players, but it appears to accept any type of entity.  The entity does not need to be in the same dimension as the player; if necessary, the player will be respawned in the right world.  If the given entity cannot be found (or isn't loaded), this packet will be ignored.  It will also be ignored if the player attempts to teleport to themselves.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2D
 | Play
 | Server
 | Target Player
 | UUID
 | UUID of the player to teleport to (can also be an entity UUID).
 |}

==== Player Block Placement ====

{| class="wikitable"
 |-
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 |rowspan="7"| 0x2E
 |rowspan="7"| Play
 |rowspan="7"| Server
 | Hand
 | VarInt Enum
 | The hand from which the block is placed; 0: main hand, 1: off hand.
 |-
 | Location
 | Position
 | Block position.
 |-
 | Face
 | VarInt Enum
 | The face on which the block is placed (as documented at [[#Player Digging|Player Digging]]).
 |-
 | Cursor Position X
 | Float
 | The position of the crosshair on the block, from 0 to 1 increasing from west to east.
 |-
 | Cursor Position Y
 | Float
 | The position of the crosshair on the block, from 0 to 1 increasing from bottom to top.
 |-
 | Cursor Position Z
 | Float
 | The position of the crosshair on the block, from 0 to 1 increasing from north to south.
 |-
 | Inside block
 | Boolean
 | True when the player's head is inside of a block.
 |}

Upon placing a block, this packet is sent once.

The Cursor Position X/Y/Z fields (also known as in-block coordinates) are calculated using raytracing. The unit corresponds to sixteen pixels in the default resource pack. For example, let's say a slab is being placed against the south face of a full block. The Cursor Position X will be higher if the player was pointing near the right (east) edge of the face, lower if pointing near the left. The Cursor Position Y will be used to determine whether it will appear as a bottom slab (values 0.0–0.5) or as a top slab (values 0.5-1.0). The Cursor Position Z should be 1.0 since the player was looking at the southernmost part of the block.

Inside block is true when a player's head (specifically eyes) are inside of a block's collision. In 1.13 and later versions, collision is rather complicated and individual blocks can have multiple collision boxes. For instance, a ring of vines has a non-colliding hole in the middle. This value is only true when the player is directly in the box. In practice, though, this value is only used by scaffolding to place in front of the player when sneaking inside of it (other blocks will place behind when you intersect with them -- try with glass for instance). 

==== Use Item ====

Sent when pressing the Use Item key (default: right click) with an item in hand.

{| class="wikitable"
 ! Packet ID
 ! State
 ! Bound To
 ! Field Name
 ! Field Type
 ! Notes
 |-
 | 0x2F
 | Play
 | Server
 | Hand
 | VarInt Enum
 | Hand used for the animation. 0: main hand, 1: off hand.
 |}

[[Category:Protocol Details]]
[[Category:Minecraft Modern]]
