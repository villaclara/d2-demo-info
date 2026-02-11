Dota2 Demo Files are stored as .dem file. Its size can vary up to 100+ Mb.

## File Structure
The File has similar structure regardless of the game played. 
- 'PBDEMS2\0' 8-byte string which indicates that this replay is replay for source 2. 
- Then 4 bytes offset. Actually after initial string there goes 8 bytes offset, but all docs in clarity and other parsers say that only 4 bytes are used as offset. Maybe other 4 bytes are used for something other, I dont know. However to actually go to next data it is needed to skip 8 bytes.
- Then go different messages.

All messages consist of: [kind][tick][size][payload]:
- kind - one of the EDemoCommands. Mostly all commands will be in range ~1-20. There is also command in that enum DEM_IsCompressed_S2 = 64 indicating that the message is compressed.
- tick - tick on the server when the message was sent
- size - size of payload
- payload - the data itself
So the general approach would be: read kind of message -> read (skip) tick, read size -> read bytes(size) to read payload -> convert payload to needed data.

Every part of message (kind, tick, size) is a varint, so they should be correctly read.

Every message could be compressed using Googles Snappy tool, so in order to parse the message correctly please do:
read [kind] -> check if it is compressed by bitwise AND with EDemoCommands.DEM_IsCompressed_S2 = 64 (64 in binary is 01000000). And while all other commands are max 20 if we have a '1' bit at 6th position then it is compressed and needs to be decompressed.

## Order of work:
1. Read magic string 'PBDEMS2\0'.
2. Skip 8 bytes.
3. First message would probably be CDemoFileHeader command. Read varint and it should be '1'.
4. After that there will be different commands.
5. When the command DEM_SyncTick = 3; appears it means that the Match data begins.
Match data doesn't start until a CDemoSyncTick message is received. Everything before this (the prologue) is not part of the visual replay. The data between CDemoSyncTick and CDemoStop is the match data, and the data after CDemoStop is the epilogue.

In match data, there are only two kinds of messages:

CDemoFullPacket (13): This packet contains the entire state of the world at that tick. Full packets are only taken once every 1800 ticks (1 minute). When random replay access is desired, Skadi uses these full packets to scan "close to" the desired tick, aggregating their information as it finds them (in order). It then follows the regular packets to the exact desired tick. From that point, only regular packets are used.

CDemoPacket (7): This contains the differences between the state of the world at this tick and the state of the world at the previous tick.

## Examples:

### Rading Varint32
```
static int ReadVarint32(BinaryReader br)
{
	int result = 0;
	int shift = 0;

	while (true)
	{
		byte b = br.ReadByte();
		//Console.WriteLine($"Currentpos - {br.BaseStream.Position}");
		//Console.WriteLine($"Byte read - {b:X}, {Convert.ToString(b, 2).PadLeft(8, '0')}");

		result |= (b & 0x7F) << shift;

		if ((b & 0x80) == 0)
			break;

		shift += 7;

		if (shift > 35)
			throw new Exception("Varint too long");
	}

	return result;
}
```

### Loop of reading messages and printing only kind of messages and its size
```
while (true)
{
	Thread.Sleep(50);
	//Console.WriteLine($"Currentpos - {reader.BaseStream.Position}");

	var nCmd = ReadVarint32(reader);
	//Console.WriteLine($"NExt cmd - {nCmd}");
	var compr = (nCmd & (int)EDemoCommands.DemIsCompressed) != 0;
	Console.WriteLine(compr);
	if (compr)
	{
		//Console.WriteLine($"Uncompressed - {nCmd & ~(int)EDemoCommands.DemIsCompressed}");
	}
	var nTick = ReadVarint32(reader);
	var nSize = ReadVarint32(reader);
	//Console.WriteLine($"Nsize - {nSize}");

	Console.WriteLine($"Currentpos - {reader.BaseStream.Position}, Cmd - {nCmd}, UCmd - {nCmd & ~(int)EDemoCommands.DemIsCompressed}, Size - {nSize}.");
	reader.ReadBytes(nSize);
}
```

### Reading CDemoFileHeader starting at 9 position
```
int cmd = ReadVarint32(reader); // EDemoCommands
if (cmd != 1) // DEM_FileHeader
{
	Console.WriteLine($"Unexpected command: {cmd}");
	return;
}
bool isCompressed = (cmd & (int)EDemoCommands.DemIsCompressed) != 0;
Console.WriteLine($"Command kind - {cmd}, iscompressed - {isCompressed}.");

int tick = ReadVarint32(reader);
// 4️⃣ Read the length of the protobuf message (varint32)
int size = ReadVarint32(reader);
Console.WriteLine($"Size - {size}");
// 5️⃣ Read payload
byte[] payload = reader.ReadBytes(size);

if (isCompressed)
{
	var command = Snappy.DecompressToArray(payload);
}

// ✅ Parse using CodedInputStream
var cis = new CodedInputStream(payload);

CDemoFileHeader demoHeader = new CDemoFileHeader();
demoHeader.MergeFrom(cis);  // this reads all fields from the stream

Console.WriteLine($"Demo Stamp: {demoHeader.DemoFileStamp}");
Console.WriteLine($"Map Name: {demoHeader.MapName}");
Console.WriteLine($"Demo Stamp: {demoHeader.DemoFileStamp}");
Console.WriteLine($"Map Name: {demoHeader.MapName}");
Console.WriteLine($"Server Name: {demoHeader.ServerName}");
Console.WriteLine($"Client Name: {demoHeader.ClientName}");
Console.WriteLine($"Patch Version: {demoHeader.PatchVersion}");
Console.WriteLine($"Game: {demoHeader.Game}");
Console.WriteLine($"Demoversionname: {demoHeader.DemoVersionName}");
```
