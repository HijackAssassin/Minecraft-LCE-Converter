# Minecraft LCE Converter

Converts Minecraft: **Legacy Console Edition** worlds between **Xbox 360**,
**PS3** and the offline **Windows 64** build — in both directions.

"Windows 64" means 4J's offline PC build of LCE. It is **not** Bedrock and
**not** Minecraft: Windows 10 Edition.

Built on [dtentiion/LCE-Save-Converter](https://github.com/dtentiion/LCE-Save-Converter),
which handles the (Xbox 360/PS3) → (Win64) direction; 

the (Win64) → (Xbox 360/PS3) direction and the emulator format are new.

---

## Running it

```bash
Minecraft LCE Converter.bat
```

or `python lce_gui.py`. Requires **64-bit Python 3.11+** and no third-party
packages.

The first screen has two buttons — **Console to Win64** and **Win64 to
Console**. After picking one, the button in the top right toggles between
**Xbox 360** and **PS3** (Xbox 360 by default), and everything below follows it.

Output goes to the `Output/` folder unless you change it.

---

## Picking an input

One button — **Select World File**. You always pick the world's *file*, and the
folder it sits in supplies the world's name and thumbnail.

| Screen | What to select |
|---|---|
| Console to Win64, Xbox 360 | the world's `.bin`, or the `savegame.dat` inside an emulator folder |
| Console to Win64, PS3 | the `GAMEDATA` inside the PS3 save folder |
| Win64 to Console | the `saveData.ms` inside the Win64 world folder |

The picker defaults to **All files**, so anything is selectable. Cancelling just
cancels — nothing reopens.

Selecting `savegame.dat` is treated as selecting the folder around it, so it
picks up `__thumbnail.png` and the folder name automatically. A `.bin` is
self-contained and uses its own embedded name and thumbnail.

---

## Save as emulator folder

Xenia and Nexia360 do not read `.bin` packages. They read a **folder** holding
the raw save and its thumbnail:

```
My World/
├── savegame.dat
└── __thumbnail.png
```

Tick **Save as emulator folder** — it sits under Output — to write that instead
of a `.bin`. It needs no template and no resigning, so for emulator use it is
the easy path. Unticked, you get the normal console format: a `.bin` STFS
package for real hardware.

The option only appears on the **Win64 to Console** screen, and only for Xbox
360. It has no meaning going the other way, and PS3 saves are always folders.

Reading is format-agnostic regardless — point *Console to Win64* at a `.bin`,
an emulator folder, or a bare `savegame.dat` and it works out which it got.

---

## Win64 world folders

A native Win64 world is a folder holding three things:

```
Parkour Map/
├── saveData.ms
├── worldname.txt              the real world name, plain UTF-8
└── thumbnails/
    └── thumbData.png
```

Converting **to** Win64 writes all three. Converting **from** Win64 reads
`worldname.txt` for the world's name and `thumbnails/thumbData.png` for its
thumbnail, and carries both to the target — into the STFS display name and
thumbnail for Xbox 360, or `SUB_TITLE` and `THUMB` for PS3.

`worldname.txt` wins over the folder name, because the folder it sits in may be
called anything. `templates/Win64 Template/` is an example: the folder is named
*Win64 Template* but the world is *Parkour Map*, and *Parkour Map* is what comes
out the other side.

---

## Thumbnails

Every LCE save has a thumbnail, and it is carried through every conversion —
resized never, re-encoded never, just copied.

| Platform | Where it lives |
|---|---|
| Xbox 360 `.bin` | STFS metadata at `0x171A` |
| Emulator folder | `__thumbnail.png` |
| Win64 | `thumbnails/thumbData.png` |
| PS3 | `THUMB` |

Whatever the input shape, the thumbnail is found and written into the target's
own slot — including into the `.bin`'s STFS metadata, replacing the bundled
header's image so the console shows *your* world, not the template's.

A 1×1 placeholder is written only if a source genuinely has no thumbnail, so the
output layout is never incomplete.

---

## Verify output

After compressing each region chunk and the main payload, the tool decompresses
it again and checks it matches byte-for-byte. It catches a bad conversion here
instead of on your console. It roughly doubles conversion time and is on by
default. Leave it on.

---

## Xbox 360: templates and signing

**A template is optional.** `templates/xbox360_header.bin` ships with the tool:
the 40 KB STFS metadata block from a real Xbox 360 save, carrying its title,
profile, console and device IDs. Without a template that block is used, so the
save comes out belonging to that console's profile.

Supply a template when building for a **different console** — pass any
Minecraft save exported from it. It is also where the target `saveVersion`
comes from; without one the tool writes **6**, which is what TU19 writes.

To change the bundled console permanently:

```bash
python -c "open('templates/xbox360_header.bin','wb').write(open('their_save.bin','rb').read(40960))"
```

**Signing is manual.** A CON save is signed with a per-console private key that
only the console has, so the tool cannot produce a valid signature. Open the
`.bin` in Horizon and use **Rehash & Resign**, then copy it to your Xbox 360.

The **hash tree** is a different thing from the signature, and the tool does
build that correctly — `validate_stfs()` walks the package exactly as Horizon's
`StfsMapNewBlock` does and checks every level. The genuine retail save is used
as the control: if the validator does not pass a real save, the validator is
wrong, not the package.

The geometry comes straight from Horizon's `StfsDevice.cs`:

```
FormatShift = ReadOnlyFormat ? 0 : 1
BlockValues = ReadOnlyFormat ? {0xAB, 0x718F} : {0xAC, 0x723A}
```

Both hang off bit 0 of the volume descriptor's **flags byte at `0x37B`**, so
that byte must be `0`. Setting it to `1` makes a reader use the other geometry
while the file is laid out with this one, and every hash lookup lands on the
wrong block — which is exactly the
`STFS: hash mismatch for block number 0x00000000:2` Horizon reports.

`ContentId` at `0x32C` is SHA1 over the metadata region `0x344`–`0xA000`, and
must be written last, after the display name, thumbnail, volume descriptor and
root hash are all in place.

If a resigned save still does not appear, set the **Profile ID** in Horizon to
match the gamertag that should own it. Emulators ignore signatures entirely.

---

## PS3 — experimental

PS3 → Win64 is the original converter and is solid. **Win64 → PS3 is new and
has not been tested on hardware.** It writes a normal PS3 save folder:

```
NPEB01899--MYWORLD/
├── GAMEDATA     uncompressed big-endian payload
├── PARAM.SFO    generated; SUB_TITLE carries the world name
└── THUMB        the world thumbnail
```

The data itself verifies clean — every region chunk round-trips to identical
RLE bytes. The gap is that a real PS3 also requires a valid **`PARAM.PFD`**,
which contains keyed hashes this tool cannot generate. Run the folder through a
PS3 save resigner before putting it on a console. RPCS3 is generally more
forgiving.

PS3 stores the world name in `PARAM.SFO`'s `SUB_TITLE`, not in `level.dat` —
`LevelName` there is always the literal `"world"`.

---

## Save version caveat

`saveVersion` 8 is `SAVE_FILE_VERSION_COMPRESSED_CHUNK_STORAGE`, where 4J
changed chunks to store compressed storage formats directly. A world genuinely
saved at version ≥ 8 stores chunks in a layout TU19 cannot read, and no
container conversion fixes that.

The tool warns when the source says ≥ 8 but still writes the target version.
Worlds produced by the console → Win64 direction always report version 9
regardless of their real contents, so for those the warning is expected and
harmless.

---

## Verification

`test_all_directions.py` exercises every path against a matched TU19 world pair
(the same world as a genuine Xbox 360 `.bin` and as a Win64 save), comparing
decoded RLE chunk data rather than just file sizes:

```bash
python test_all_directions.py
```

| Path                     | Plain files | Region chunks |
| ------------------------ | ----------- | ------------- |
| Xbox `.bin` → Win64      | 133 / 133   | 3564 / 3564   |
| Win64 → Xbox `.bin`      | 133 / 133   | 3564 / 3564   |
| Win64 → emulator folder  | 133 / 133   | 3564 / 3564   |
| Win64 → PS3              | 133 / 133   | 3564 / 3564   |
| PS3 → Win64 (round-trip) | 133 / 133   | 3564 / 3564   |

It reads its sample worlds from `samples/`.

---

## How it works

Console and Win64 saves hold the same payload; only the container differs.

|                      | Xbox 360           | PS3                           | Win64         |
| -------------------- | ------------------ | ----------------------------- | ------------- |
| Container            | STFS CON `.bin`    | folder                        | `saveData.ms` |
| Payload compression  | XMemCompress / LZX | none                          | zlib          |
| Endianness           | big                | big                           | little        |
| Region chunk payload | LZX                | `[u32 BE size]` + raw deflate | zlib          |

File *contents* are never byte-swapped — LCE NBT is big-endian on every
platform — so only container fields change. Chunk data is decompressed to 4J's
RLE stream and recompressed for the target; the RLE bytes themselves are never
touched.

`savegame.dat`:

```
[0x00] u32 BE  payloadLen = 8 + compressedLen
[0x04] u32 BE  0                  <- the loader requires this to be zero
[0x08] u32 BE  decompressedSize
[0x0C] ...     LZX data
```

Compression parameters, from `Minecraft.World/compression.cpp`:

```c
params.Flags = 0;
params.WindowSize = 128 * 1024;
params.CompressionPartitionSize = 128 * 1024;
```

`CompressionPartitionSize` matters — the 512 KB default produces a stream the
console will not decode.

---

## Files

```
LCE Converter/
├── Minecraft LCE Converter.bat   launcher
├── lce_gui.py                    the interface
├── lce_engine.py                 all four conversion directions
├── converter.py                  Xbox 360 -> Win64 (upstream)
├── converter_ps3.py              PS3 -> Win64 (upstream)
├── test_all_directions.py        verification
├── README.md
├── xcompress64.dll               Microsoft XCompress (LZX)
├── chm_lzx.dll                   CHMLib LZX decoder
├── LZXDecompression.dll          LDI decoder
├── templates/
│   ├── xbox360_header.bin        bundled console identity
│   └── Win64 Template/           example of a native Win64 world folder
├── samples/                      worlds the test suite runs against
│   ├── TU19 Xbox 360 World Template.bin
│   ├── TU19 Win64 World Template/
│   └── Emulation World Format Example/
└── Output/                       converted saves land here
```

Everything at the root is required to run the tool. `templates/` and `samples/`
are the only extras: `xbox360_header.bin` is used at runtime, and the rest is
used by `test_all_directions.py`.

The formats were worked out from the 4J LCE source (`Minecraft.World/`) and the
Horizon STFS implementation. Those trees are no longer part of the project — the
findings are implemented in `lce_engine.py`, with the source file and field each
value came from noted in the comments beside it.
"# Minecraft-LCE-Converter" 
