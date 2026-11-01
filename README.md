# mc-changes

Small files that tell tools **which files of a Minecraft resource pack are the same as the game's own files**,
for many game versions of Java Edition and Bedrock Edition. A tool can then leave those files out of a pack:
the game loads its own copy anyway, so the pack gets smaller and nothing changes in the game.

The files hold **hashes (fingerprints) of the game files, not the game files**. They are made by
`build_mcchanges_jar.py` (see `tools/`). [PackSquash (fork)](https://github.com/mocny-gaz/PackSquash-uv12) reads them
with the options `mcchanges_path` and `mcchanges_version`. The format is documented below, so that other projects
can read them too.

> Not an official Minecraft product. Not approved by or associated with Mojang or Microsoft.
> The files hold no game files, only hashes. This is not legal advice: read the license of this repository.

## Download

Get the files from the [Releases](../../releases) page, not from the git history:

| File | What |
|---|---|
| `mcchanges-java.jar` | Java Edition, every release from 1.6.1 |
| `mcchanges-bedrock.jar` | Bedrock Edition, the newest releases |

Direct link of the newest file: `https://github.com/mocny-gaz/mc-changes/releases/latest/download/mcchanges-java.jar`

Files named `old_mcchanges-*.jar` use the old layout (bigger). They still work with PackSquash, but you do not need them.

## Use it in another project

A reader needs about 100 lines. `example_reader.py` (Python, standard library only) is a complete example:

    python example_reader.py mcchanges-java.jar 1.21.4 assets/minecraft/lang/en_us.json my_en_us.json

It prints how many entries of your language file are the same as the game's. Read the format below.

### The format

A jar is a zip file. `mcchanges.json` in the root is the index:

    {
      "format": 1,
      "kind": "hashes",                  "hashes" (no game files) or "full" (the files are stored)
      "layout": "compact",               (hashes jars) language keys are stored as langfull/langdelta
      "edition": "java",                 "java" or "bedrock"
      "versions": ["1.6.1", "1.6.2", ...],     oldest first
      "removed": { "1.7.2": ["assets/..."] },  files that this version deleted compared with the version before
      "hashed":  { "1.6.1": { "assets/minecraft/textures/blocks/stone.png": "png:16x16:ab12..." } }
    }

Every version is compared with the version before it, and `hashed[version]` only lists what that version added or
changed. To find a file `P` in version `V`: go through the versions from `V` down to the oldest. In each one:

1. `hashed[version][P]` exists: this is the token of the file. Stop.
2. A file `<version>/P` is in the zip (full jars only): this is the file. Stop.
3. `P` is in `removed[version]`: the game does not have the file. Stop.

If you reach the oldest version, the game does not have the file (or it is not tracked, for example code).

**Tokens** (`algorithm:digest`, digests in lowercase hexadecimal). A token that you do not understand never matches:
keep the file. A wrong "no match" only costs size. A wrong "match" removes a file that is needed, so never guess.

| Token | The file matches if |
|---|---|
| `sha256:<hex>`, `sha1:<hex>` | the digest of its bytes is the one in the token. Sounds use `sha1:` (from the Mojang asset index) |
| `json:<hex>` | the SHA-256 of its canonical JSON is the one in the token. Parse it as UTF-8 (a BOM is ignored), then write it with no spaces, object keys sorted by their UTF-8 bytes, strings with `"` `\` and control characters escaped (like `json.dumps(ensure_ascii=False)`), integers and floats as the shortest text that reads back the same. NaN and Infinity are invalid. Very small or big decimal numbers may be written differently by other languages: then the file does not match |
| `png:<w>x<h>:<hex>` | it is a PNG of the same size, and the SHA-256 of its pixels is the one in the token. The pixels are RGBA, 8 bits per channel, row by row from the top, with the transparency chunk applied, gray images of 1, 2 and 4 bits scaled up (x255, x85, x17), and the red, green and blue bytes of fully transparent pixels set to 0. Images that are 16 bit, interlaced, damaged, or bigger than 4096 x 4096 pixels are never `png:` tokens: they have `sha256:` |
| `langfull:<hex>`, `langdelta:<hex>` | see below. Only for language files (`assets/<namespace>/lang/*.json` and `*.lang`) |

**Language files.** The token does not say if a whole file matches: it lets you remove the entries that are the same
as the game's. Next to the version files there is `<version>/<path>.langkeys`. `<hex>` is the SHA-256 of that file.
Check it. If it is wrong, do not use the file.

- The hash of one entry is the first 8 bytes of SHA-256(`key` + `0x00` + `value`). In JSON files the value is the canonical
  JSON of the value, quotes included (`"One"`). In `.lang` files it is the text after the first `=`; lines are split at `\n`,
  trailing `\r` and `\n` are removed, blank lines, lines starting with `#` and lines without `=` are ignored, and the last
  value of a repeated key counts.
- `langfull`: the file is the 8 byte hashes of all entries, one after the other, sorted.
- `langdelta`: the file is a little endian 32 bit number N, then N added hashes, then the removed hashes (the rest of
  the file). They change the list of the last version that had the same file. Follow the layers down to the next one
  that has the path in `hashed` or `removed`: it must be a `langfull` (start from its list) or a `langdelta`
  (continue down). Then apply the changes from the oldest to the newest. A `removed` entry in the chain, a
  missing file, or a wrong digest means: do not use it.
- An entry of a pack file is "the same as the game's" if its hash is in the final list. Remove those entries, and
  drop the file if no entry is left. (PackSquash does this with the option `skip_language_keys_identical_to_mcchanges`.)
- Old jars use `langkeys:<hex>`: the file holds 12 byte hashes of all entries (the same formula, longer).

### Safety rules

- Match only on the exact rules above. When in doubt, keep the file.
- Never use a `.langkeys` file with a wrong digest.
- The hashes jars do not hold Mojang's files. A `kind: "full"` jar does, so never share that kind.

## Use it with PackSquash (fork)

1. Use the fork:
   <https://github.com/mocny-gaz/PackSquash-uv12/releases>
2. Put the jar somewhere, for example `C:\amc\mcchanges-java.jar`.
3. Make `packsquash.toml` (use single quotes for Windows paths):

       pack_directory = 'C:\packs\my_pack'
       output_file_path = 'C:\packs\my_pack.zip'
       mcchanges_path = 'C:\amc\mcchanges-java.jar'
       mcchanges_version = '1.21.4'

4. Run it:

       packsquash.exe packsquash.toml

PackSquash leaves out the files of your pack that are the same as the files of game version `1.21.4`: textures with the
same pixels, JSON files with the same contents, sounds that are exact copies, and lines of language files with the same
value. Everything else is squashed as usual. `mcchanges_version` must be the game version that the pack is for.

Example: a pack that only changes 20 textures and one language line, built on top of a full copy of the vanilla assets,
gets the size of those changes.
