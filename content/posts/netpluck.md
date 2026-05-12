---
title: "netpluck: A Tool to Scan and Pluck Remote Archives"
date: 2026-05-11T21:44:06-06:00
draft: false
tags: [python,byterange,pip]
---

## Reading Remote Zip Archives Without Downloading the Entire File

At my current job we manage large quantities of large archive files. Many of these files are quite large, frequently ranging in the hundreds of megabytes to several gigabytes. Our storage archives contain over a petabyte of data and expands on a daily basis. A project came up where I needed a way to verify the contents of these files for validation and also to retrieve a few small files inside the compressed archives. At first I would pull the entire archive down, unpack the archive, catalog the files, and store the file I needed from the archive. For such large files this was both expensive in terms of bandwidth but also disk space and memory when doing many of these in parallel. I wondered if it was possible to scan the archive and retrieve the file I needed from each archive without actually fetching the entire thing. The archives were in zip format, typically using Deflate as the compression method.

After looking up the [zip archive format](https://en.wikipedia.org/wiki/ZIP_(file_format)), one weekend, I noticed it contained headers that let you seek through the archive without having to decompress the entire archive. I wanted to build a tool that solved this while not relying on none core python libraries. On it's own this isn't super useful; what I needed in addition to this was a way to fetch only subsets of files off remote locations. In HTTP there is a header [Range](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Range) that let's you request a subset of a response. This is very useful when resuming downloads or threading downloads. In addition to the Range header, you can also get the [Content-Length](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Length) which tells you the response size. I combined this into a *Virtual File* class that let me pretend the file was already in memory and just seek and read the bytes I cared about. Combining all these things allowed me to get a url from our content delivery network bucket system and then query the url and fetch both the table of contents, ToC, and pluck individual files I wanted out of the archive. This significantly lowered the costs of egress fees to scan these files (often by 99%) and greatly sped up our independent file validation tools.

This is now available as a python tool at https://github.com/jjanzer/netpluck or by simply installing via pip: 

```bash
pip install netpluck
```

I have since expanded it to support local files, HTTP/HTTPs, "Buckets" (explained below) along with adding support for Tar files (if not compressed) in addition to Zip files.

## Using NetPluck

This demonstrates how to list all files in the remote zip file along with fetching json files containing the word *metadata* over a Backblaze B2 private bucket. Using the `--stats` flag shows 4 lookups totalling a combined payload size of less than 1 MB even though the entire file is ~1.6GB.

```bash
$ time netpluck.exe --toc --path="b2://bar/foo/large_file.zip" --config=../configs/config.b2.json --filter=".*metadata.*\.json" --stats --out ./extracted/
Found file: bar/foo/large_file.zip, Size: 1695519682
BodyShape.dsf
fig_029079.json
fig_029079.tip.png
fig_029079.txt
HeadShape.dsf
manifest.json
metadata.json
metadata_full.json
metadata_split.json
textures/Arms_R_1004_ggf58l.png
textures/Arms_SO_1004_ya20eh.png
textures/SuperheroBoots/S_Boots_Green_Metallic_amtaf.png
... snipped ...
textures/SuperheroBoots/S_Boots_Green_Normal_5jhnb3.png
textures/SuperheroBoots/S_Boots_Green_Roughness_57t2dg.png
textures/Eyebrows/9Brows_OP_1prs9z.png
textures/Eyebrows/9Hair_BM_13o49y.png
textures/Eyebrows/9Hair_DF_52443d_9ffn3.png
textures/Eyebrows/9Hair_LW_3sa1lm.png
textures/Eyebrows/9Hair_SP_c7b079_6zqu7g.png
textures/Eyebrows/9Hair_SP_f9f9f9_6zqu7g.png
[1/3]  33.33% metadata.json => extracted/metadata.json
[2/3]  66.67% metadata_full.json => extracted/metadata_full.json
[3/3] 100.00% metadata_split.json => extracted/metadata_split.json

File size: 1.58GB
Cache hits: 123 size: 773.56KB
Uncached reads: 4 size: 73.95KB
Bytes saved: 1.58GB 100.00%

real    0m1.534s
user    0m0.000s
sys     0m0.000s
```

## Abstracting the Remote Fetch into Bucket Handler

I've since written my own bucket handler abstraction tool available at: https://github.com/jjanzer/buckethandler or via pip:

```bash
pip install buckethandler
```

This tool makes handling of "bucket" systems such as Backblaze B2, Amazon S3, or other file stores such as Dropbox extremely easy. When combined with netpluck you can use it as the engine to query your files directly through APIs.

```bash
$ bh ls b2:// --config=../configs/config.b2.json
bar/car/sim/a.txt                               text/plain                      4.00B   2026-03-18 20:56:42
bar/car/sim/b.txt                               text/plain                      4.00B   2026-03-18 20:56:42
bar/car/sim/micro.zip                           application/x-zip-compressed    350.00B 2026-03-18 20:56:42
bar/car/sim/nested/a/b/c.txt                    text/plain                      0B      2026-03-18 20:56:42
bar/car/sim/sample_data.zip                     application/x-zip-compressed    1.52MB  2026-03-18 20:56:42
bar/car/sim/sample_data_uncompressed.zip        application/x-zip-compressed    4.21MB  2026-03-18 20:56:42
bar/car/sim/test.zip                            application/x-zip-compressed    3.29MB  2026-03-18 20:56:42
bar/foo/a.txt                                   text/plain                      0B      2026-03-28 14:30:31
bar/foo/b.txt                                   text/plain                      0B      2026-03-28 14:30:31
bar/foo/large_file.zip                          application/x-zip-compressed    1.58GB  2026-04-04 14:14:06
bar/foo/music.mp3                               audio/mpeg                      0B      2026-03-28 14:30:31
bar/foo/s3_hw.txt                               text/plain                      12.00B  2026-04-04 14:38:56
bar/foo/sim.tar                                 binary/octet-stream             9.02MB  2026-04-03 11:32:19
bar/music.mp3                                   audio/mpeg                      0B      2026-03-28 14:31:08
foo/a.txt                                       text/plain                      0B      2026-03-28 14:36:14
foo/b.txt                                       text/plain                      0B      2026-03-28 14:36:14
foo/music.mp3                                   audio/mpeg                      0B      2026-03-28 14:36:14
===========================================================================================================
Files: 17, minTime: 2026-03-18 20:56:42 maxTime: 2026-04-04 14:38:56 time delta: 16:17:42:14
```
