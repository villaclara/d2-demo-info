Demo File Format - skadistats/skadi GitHub Wiki
The .dem file from Valve are very simple files. They consist of:

8-byte header "PBUFDEM\0"
A 4-byte word with an unsigned int offset
this offset is the offset to a summary protobuf message containing information about the .dem file
The remainder of the file is a series of messages, each one encoded as:
kind - a varint* specifying the message kind. The 'kind' values can be found in /protobuf/demo.proto, and only a subset show up in actual replay files.
tick - a varint specifying the tick of the following message. You can think of ticks as "frames" in the .dem, pointing to specific parts of the replay.
size - a varint specifying the size of the message in bytes.
pbmsg - the actual message itself.
The messages repeat until the replay has ended. If you'd like to see what kind of information is included in the Demo.game_info (meta-information about the game) See Demo.file_info

*a varint is a variable-length integer, as defined in protobuf's spec.

Match data doesn't start until a CDemoSyncTick message is received. Everything before this (the prologue) is not part of the visual replay. The data between CDemoSyncTick and CDemoStop is the match data, and the data after CDemoStop is the epilogue.

In match data, there are only two kinds of messages:

CDemoFullPacket: This packet contains the entire state of the world at that tick. Full packets are only taken once every 1800 ticks (1 minute). When random replay access is desired, Skadi uses these full packets to scan "close to" the desired tick, aggregating their information as it finds them (in order). It then follows the regular packets to the exact desired tick. From that point, only regular packets are used.

CDemoPacket: This contains the differences between the state of the world at this tick and the state of the world at the previous tick.
