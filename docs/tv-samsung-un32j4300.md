# The living room TV: Samsung UN32J4300AG (DLNA)

What this TV plays directly, what forces Jellyfin to transcode, and how to test any of it without
walking to the living room and turning the set on.

> This repo describes the old Docker stack. The server today runs as a native NixOS service
> (`dotfiles/system/services/jellyfin.nix`), but everything below holds for both: the DLNA
> profile decides, not the way Jellyfin was installed.

## Identification

![The TV's label](img/tv-samsung-un32j4300-etiqueta.jpg)

| Field       | Value                                          |
| ----------- | ---------------------------------------------- |
| Model       | UN32J4300AG                                    |
| Type No.    | UN32J4300                                      |
| Model Code  | UN32J4300AGXZD                                 |
| Version No. | CA07                                           |
| Power       | DC 19V, 39W                                    |
| ANATEL      | 0187-15-3269                                   |
| Built at    | Polo Industrial de Manaus (Samsung Amazônia)   |

> The serial number, barcode and QR code are blacked out in the picture, because this
> repository is public.

A 2015 J-series set with an HD panel and an old Tizen build. It has **no usable app store**, so the
Jellyfin app is not an option: DLNA is the only way in.

Over DLNA it announces itself as **UN32J4303** (not `...4300`, as the label reads) and Jellyfin
matches it to the built-in **"Samsung Smart TV"** profile.

## What the TV accepts, measured and not guessed

Tested against Jellyfin 10.11.11 with DLNA plugin 11.0.0.0, querying `ContentDirectory` with the
TV's own User-Agent so the server applies the Samsung profile:

```text
DLNADOC/1.50 SEC_HHP_[TV] UN32J4303/1.0
```

Four 30-second clips cut from one source file, changing one variable at a time:

| Clip           | Container | H.264 level | Result          |
| -------------- | --------- | ----------- | --------------- |
| A              | MKV       | 4.2         | transcode       |
| B              | MP4       | 4.2         | transcode       |
| C              | MP4       | **4.1**     | **direct play** |
| D              | MKV       | **4.1**     | **direct play** |

**The level is the only thing that matters. The container does not.** MKV plays directly as long as
the video is Level 4.1 or lower, which means a Level 4.2 file usually needs nothing but a relabel
of the bitstream, not a re-encode:

```bash
ffmpeg -i in.mkv -map 0 -c copy -bsf:v h264_metadata=level=41 out.mkv
```

A 3 GB file goes through that in about 12 seconds and keeps every audio and subtitle track.

HEVC is the one case with no shortcut: the TV cannot decode it, so it has to be re-encoded to
H.264.

### Transcoding costs the pause button, not just seeking

Direct play advertises `DLNA.ORG_OP=01` (byte seeking). A live transcode advertises no `OP` at all,
i.e. `OP=00`, and there is no way to ask the server to resume from a given position. On this TV,
**pausing is as unreliable as seeking** on a transcoded stream: the set stops reading the socket
and, with no byte-range support, tends to restart the stream from zero.

If a title is going to be watched more than once, converting it once to a direct-play format beats
transcoding it on every play.

### Transcoding runs on the GPU

Confirmed by inspecting the running ffmpeg's command line: VAAPI decode plus QSV encode on the
Intel Arc B580, the whole pipeline on the GPU with no copy back to the CPU.

```text
vaapi=va:/dev/dri/renderD128,driver=iHD   qsv=qs@va
-hwaccel vaapi -hwaccel_output_format qsv
scale_vaapi -> hwmap=derive_device=qsv -> h264_qsv
```

## Audio: the TV gets whatever is flagged `default`

Over DLNA the server picks the track, not the TV, and it sends the one flagged `default` in the
file. A **DUAL** release with pt-BR as the default track plays dubbed with no interaction at all,
which is the most reliable setup here.

**E-AC3 direct plays on this TV**: it is in the Samsung profile, verified with a real file.

The trap is not the codec, it is **two tracks flagged `default` at once**. When that happens
Jellyfin resolves the ambiguity on its own and can pick the wrong language. Check before importing:

```bash
ffprobe -v error -show_entries stream=index,codec_type,codec_name:stream_tags=language,title \
  -show_entries stream_disposition=default -of default=noprint_wrappers=1 file.mkv
```

Fixing the flags is a remux, not a re-encode, around 10 seconds for a 3 GB file:

```bash
ffmpeg -i in.mkv -map 0 -c copy \
  -disposition:a:0 default -disposition:a:1 0 \
  -disposition:s:0 default+forced -disposition:s:1 0 out.mkv
```

## Subtitles

Jellyfin exposes subtitles to Samsung sets through the proprietary `<sec:CaptionInfoEx>` tag, which
this generation reads. With several tracks it emits several tags and **the TV normally uses only
the first one**.

During playback the subtitle usually does not turn itself on: **Tools > Subtitle**.

A direct-played file with **embedded** subtitles shows zero exposed captions, and that is correct:
the subtitle rides inside the container and the TV reads it from there. Only external `.srt`
sidecars need `CaptionInfoEx`.

### The `pob` vs `pt-br` trap

In `SubtitleDownloadLanguages` (library options) use **`pt-br`**, not `pob`.

Both work as a search token in the OpenSubtitles plugin, but Jellyfin registers the downloaded
track as `pt-br`. With the option set to `pob` the comparison never matches, so the server believes
the subtitle is missing and **re-downloads it on every refresh**, producing `.pob.0.srt`,
`.pob.1.srt` and so on, and burning the free OpenSubtitles account's daily quota (20/day).

External subtitle names Jellyfin understands, next to the video:

```text
Movie Name (Year).pt-br.srt
Movie Name (Year).pt-br.default.srt     # flags it as the default
```

Write them as **UTF-8 without BOM** and with **CRLF** line endings: this generation of Samsung
firmware chokes on a BOM.

## The DLNA plugin cannot list a container's children

Any query that walks by `ParentId` comes back empty on plugin v11. That breaks:

- `movies_` (the "Movies" entry inside a movie library)
- `series_` (the "Series" entry inside a TV library)
- every **series** and **season** container

Not a configuration problem: the libraries have the right `CollectionType`, my account is an
admin with `EnableAllFolders`, the same queries return everything through Jellyfin's own API, and it
survives both a full scan and a service restart. v11 is already the newest release.

What still works: **Latest**, **Genres**, and drilling into a genre.

### For movies this is survivable

Latest and Genres list movies as playable items, so the TV reaches them. Only the "Movies" submenu
is dead.

### For series it is fatal, and needs a workaround

Latest lists the series as a *container*, and that container does not open, so the episodes are
unreachable. `BrowseMetadata` on an episode does return a valid stream URL, which confirms the
files are serveable and only the *listing* is broken.

The way around it is a second library of type `movies` holding one folder per episode:

```text
/srv/media/media/Episodios/
├── Dark S02E01 - Começos e fins/Dark S02E01 - Começos e fins.mkv
└── Dark S02E02 - Matéria escura/Dark S02E02 - Matéria escura.mkv
```

Reflink the files (`cp --reflink=always`) so the second copy costs no disk. Turn the metadata
fetchers **off** for that library, otherwise Jellyfin matches each episode against a TMDB movie and
pulls in wrong posters and plots. The proper `tvshows` library stays in place for the web UI, where
seasons, titles and watch progress all work.

Latest sorts newest-first, so episodes come out backwards. Rewriting each item's
`DateCreated`, giving the first episode the most recent stamp, puts them back in watch order.

## Seeking can reboot the TV: keep the MKV lean

On 10/10/2026 the TV started **rebooting itself** whenever I skipped forward or back in Stranger
Things season 5. Season 4 of the same show never did it, and neither did Dark.

The server was not the problem. Every episode was direct play (`Static=true`, `DLNA.ORG_OP=01`),
the byte-range requests a seek produces came back `206` from any offset, and no `ffmpeg` was
running. The crash is inside the TV's own player, after the jump.

The obvious suspects were ruled out by comparing against files that seek fine:

| Check                       | Season 5        | Dark / season 4 |
| --------------------------- | --------------- | --------------- |
| Cues (the seek index)       | present, at end | present, at end |
| Keyframe interval           | 1.7s to 3.2s    | 4.8s            |
| Codec / H.264 level         | H.264 L4.0      | H.264 L3.1-4.0  |

What did differ was how the files were muxed, and season 5 came from two different releases:

- **E01 to E04 carried 46 subtitle tracks**, every language the source had. A seek makes the
  player resynchronize every track at the new position.
- **E05 to E08 carried a PNG cover as an attachment**, which ffprobe reports as a second video
  stream (`attached_pic`).

Season 4 had neither: one video, two audio tracks, two subtitles.

**The fix** was a remux (stream copy, no re-encode, duration identical to the millisecond) down to
that same lean layout: one video, the pt-BR and English audio, and only the two pt-BR subtitles
(forced and full), with no attachments and no cover. After it, seeking stopped rebooting the TV.

```bash
ffmpeg -i in.mkv -map 0:v:0 -map 0:a -map 0:<forced-pt-br> -map 0:<full-pt-br> -c copy \
  -map_metadata 0 -map_chapters 0 \
  -disposition:a:0 default -disposition:a:1 0 \
  -disposition:s:0 default+forced -disposition:s:1 0 out.mkv
```

**What is NOT known is which of the two caused it.** Both changes went in at once, and the TV
keeps no log. My strongest guess is the 46 subtitle tracks: a 2015 set with little RAM resyncing
that many streams is the kind of thing that runs out of memory, and running out of memory ends in a
reboot rather than an error message. The cover is the weaker suspect. Telling them apart would take
fixing one group and leaving the other as it was.

The rule I keep from it regardless: **import into the lean layout from the start**, one video, the
audio tracks, pt-BR subtitles only, no attachments.

## Testing without walking to the TV

### Can the TV see the server? (SSDP discovery)

```bash
python3 - <<'EOF'
import socket
MSG = "\r\n".join(['M-SEARCH * HTTP/1.1','HOST: 239.255.255.250:1900',
  'MAN: "ssdp:discover"','MX: 3','ST: urn:schemas-upnp-org:device:MediaServer:1','',''])
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM); s.settimeout(4)
s.sendto(MSG.encode(), ('239.255.255.250', 1900))
try:
    while True:
        data, addr = s.recvfrom(2048)
        for line in data.decode('utf8','replace').splitlines():
            if line.lower().startswith(('location','server')):
                print(addr[0], line)
except socket.timeout:
    pass
EOF
```

`LOCATION` has to point at the LAN address (`192.168.1.10`). A Docker bridge address (`172.x`) or a
VPN tunnel address means the TV will not reach the server. See the `VirtualInterfaceNames` note in
`dotfiles/docs/notes/services/jellyfin.md`.

### What the TV gets for each title

```bash
UUID=$(curl -s http://192.168.1.10:8096/System/Info/Public | python3 -c 'import sys,json;print(json.load(sys.stdin)["Id"])')
UUID=$(python3 -c "import sys;u=sys.argv[1];print(f'{u[:8]}-{u[8:12]}-{u[12:16]}-{u[16:20]}-{u[20:]}')" "$UUID")
CTL="http://192.168.1.10:8096/dlna/$UUID/contentdirectory/control"

curl -s -X POST "$CTL" \
  -H 'Content-Type: text/xml; charset="utf-8"' \
  -H 'SOAPAction: "urn:schemas-upnp-org:service:ContentDirectory:1#Browse"' \
  -A 'DLNADOC/1.50 SEC_HHP_[TV] UN32J4303/1.0' \
  -d '<?xml version="1.0"?><s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/"><s:Body><u:Browse xmlns:u="urn:schemas-upnp-org:service:ContentDirectory:1"><ObjectID>0</ObjectID><BrowseFlag>BrowseDirectChildren</BrowseFlag><Filter>*</Filter><StartingIndex>0</StartingIndex><RequestedCount>20</RequestedCount><SortCriteria></SortCriteria></u:Browse></s:Body></s:Envelope>'
```

In the response, per item:

- `Static=true` in the URL → **direct play**
- `DLNA.ORG_CI=1` → **transcode**
- `DLNA.ORG_OP=01` → seeking and pausing work
- `<sec:CaptionInfoEx>` → a subtitle is being offered to the TV
- `AudioStreamIndex=N` → cross-check `N` against the item's `MediaStreams` to see which language
  actually goes out

Swap `ObjectID` for `latest_<library-id>` to list real titles.

### What the TV exposes for debugging

Very little. The TV answers on the LAN (`192.168.1.20`, DHCP name `TV-Samsung-Sala`) with only two
TCP ports open:

| Port | What it is                                                             |
| ---- | ---------------------------------------------------------------------- |
| 7676 | UPnP media renderer (`AllShare1.0`): AVTransport, RenderingControl     |
| 8080 | DIAL, the API the YouTube app uses to cast to it                       |

No telnet, no ssh, no `sdb` (26101), so there is **no way to read the TV's own logs**. The server
is the only side that can be observed.

One useful trick: the TV never syncs its clock, so the `Date` header of any reply is really its
**uptime since power-on**. A value like `Thu, 01 Jan 1970 00:37:20 GMT` means it booted 37 minutes
ago, which makes it a cheap reboot detector:

```bash
curl -s -m 3 -i http://192.168.1.20:8080/ | grep -i '^date:'
```

The router (OpenWrt) sees the TV in its DHCP leases but has no `tcpdump`, and its log buffer did not
keep the DHCP renewals a reboot would leave. To capture what the TV asks for during playback, the
right place is the server itself, since it is one end of the stream:

```bash
sudo tcpdump -i enp7s0 -w tv.pcap host 192.168.1.20 and port 8096
```

### Does it actually stream?

```bash
curl -s -A 'DLNADOC/1.50 SEC_HHP_[TV] UN32J4303/1.0' \
  -r 1000000000-1000200000 -o /tmp/probe.bin -w '%{http_code}\n' "<stream URL>"
```

`206` proves byte ranges work, which is the mechanism behind pause and seek. Checking that no
`ffmpeg` process appears meanwhile proves it is really direct play.

## Not verified

The measurements above were taken **on the server side**: discovery, browsing, the playback
decision, and the stream arriving decodable with the right audio track. Real playback on the set
has since confirmed the practical parts, direct play, dubbed audio by default, pause and seeking.
The panel resolution and the absence of a Jellyfin app still come from the J-series spec sheet,
not from testing the set, and the cause of the seek reboot is narrowed to two suspects, not proven.
