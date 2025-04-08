---
layout:     post
title:      "Leafing through CVMFS Catalogs"
date:       2025-04-08 16:31:16 -0500
tags:       cvmfs firewall
---
I'm mainly writing this down so I remember how to dig into these bits of CVMFS internals next time[^1].

As has happened a couple of times, I discovered today that the campus firewall was blocking one specific file on the
[CVMFS][cvmfs] Stratum 0 *and* Stratum 1 servers hosted inside the campus network. Fetching the file from inside the
network works as expected, but from outside it fails:

```console
[user@stratum1 ~]$ cvmfs_server snapshot repo.example.org
Replicating from catalog at /gtdb
  Processing chunks [8056 registered chunks]: ........
failed to download http://stratum0.example.org/cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P (15 - host serving data too slowly)
couldn't reach Stratum 0 - please check the network connection
terminate called after throwing an instance of 'ECvmfsException'
  what():  PANIC: /home/sftnight/jenkins/workspace/CvmfsFullBuildDocker/CVMFS_BUILD_ARCH/docker-x86_64/CVMFS_BUILD_PLATFORM/cc9/build/BUILD/cvmfs-2.11.5/cvmfs/swissknife_pull.cc : 286
Download error
```

Additional testing with curl shows that no bytes are returned for the request and eventually the connection times out:

```console
[user@stratum1 ~]$ curl -sv \
    http://stratum0.example.org/cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P >/dev/null
*   Trying 192.0.2.42...
* TCP_NODELAY set
* Connected to stratum0.example.org (192.0.2.42) port 80 (#0)
> GET /cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P HTTP/1.1
> Host: stratum0.example.org
> User-Agent: curl/7.61.1
> Accept: */*
>
  0     0    0     0    0     0      0      0 --:--:--  0:10:13 --:--:--     0* Recv failure: Connection timed out
  0     0    0     0    0     0      0      0 --:--:--  0:10:14 --:--:--     0
* Closing connection 0
curl: (56) Recv failure: Connection timed out
```

Upon describing the problem for the firewall admins, they sent back a few lines from the log showing the denial: the
file type (or "application" in Palo Altese) was detected as `flash` (as in, Adobe/Shockwave Flash). Why is my CVMFS repo
full of public genomic data serving up Adobe Flash?

Well, it's not. CVMFS is a chunked, content-hash-addressed filesystem. I had recently published the [Genome Taxonomy
Database][gtdb] and suspected that the file in question (`2104e02468491fa4e61599c176b16c98a052d4P`) to be a chunk from
that rather large database. But it's best to confirm such suspicions.

To do that, I need to know what file the chunk belongs to.

# Locate the revision that added the chunk

There's probably a more clever way to do this but I started with the mod date on the hash file:

```console
[user@stratum0 ~]$ ls -lh /srv/cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P
-rw-r--r-- 1 user user 2.4M Apr  4 14:04 /srv/cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P
```

This should point to the revision/tag (unless you have a lot of them around that time):


```console
[user@stratum0 ~]$ cvmfs_server tag repo.example.org | grep -B1 '4 Apr 2025'
stuff | 41 | 1 Apr 2025 12:34:05  |        | Stuff
gtdb  │ 42 │ 4 Apr 2025 14:09:38  │        │ GTDB database

```

# Identify files changed in that revision

The diff subcommand shows me what files were added, removed, or changed:


```console
[user@stratum0 ~]$ cvmfs_server diff -s stuff -d gtdb repo.example.org
d(# regular files): 10
d(# symlinks): 0
d(# directories): 1
d(# catalogs): 1
/ modify directory [link-count, timestamp]
/gtdb add directory +4096 bytes
/gtdb/GTDB add file +33731108154 bytes
/gtdb/GTDB_h add file +13565000049 bytes
/gtdb/GTDB.index add file +2601811970 bytes
/gtdb/GTDB.dbtype add file +4 bytes
/gtdb/GTDB.lookup add file +3621260940 bytes
/gtdb/GTDB.source add file +786640 bytes
/gtdb/GTDB.version add file +28 bytes
/gtdb/GTDB_h.index add file +2559366433 bytes
/gtdb/GTDB_mapping add file +1562822676 bytes
/gtdb/.cvmfscatalog add file +0 bytes
/gtdb/GTDB_h.dbtype add file +4 bytes
/gtdb/GTDB_taxonomy add file +9978672 bytes
```

# Locate the catalog for the offending chunk

Knowing what's changed, now I can locate the catalog corresponding to those changes:

```console
[user@stratum0 ~]$ cvmfs_server list-catalogs -h repo.example.org
218f6ec14c5f8cf2028d9f419e57157f0dbe8d2d /
├─ ae762dea060a55e6d1d075168bbbf8cd440ca6b2 /gtdb
...
```

# Extract the catalog and verify the filename

With the catalog identified, it needs to be extracted:

```console
[user@stratum0 ~]$ cat /srv/cvmfs/repo.example.org/data/ae/762dea060a55e6d1d075168bbbf8cd440ca6b2C \
    | cvmfs_swissknife zpipe -d > /tmp/catalog.sqlite
```

I can then verify that the chunk belongs to the `GTDB` file:

```
[user@stratum0 ~]$ sqlite3 /tmp/catalog.sqlite
SQLite version 3.34.1 2021-01-20 14:10:07
Enter ".help" for usage hints.
sqlite> SELECT c.name
FROM chunks ch
JOIN catalog c
  ON ch.md5path_1 = c.md5path_1 AND ch.md5path_2 = c.md5path_2
WHERE ch.hash = X'ae762dea060a55e6d1d075168bbbf8cd440ca6b2';
GTDB
```

# Check file contents

But why stop there? A little digging suggests that the file magic for a Flash file should be ASCII `CWS`, `FWS`, or
`FLV`, so let's confirm:

```console
[user@stratum0 ~]$ cat /srv/cvmfs/repo.example.org/data/73/2104e02468491fa4e61599c176b16c98a052d4P \
    | cvmfs_swissknife zpipe -d | head -c 3 ; echo
CWS
```

And there we are, a Flash application, according to Palo Alto. Viewing the rest of the contents of the chunk also
confirm that the data after `CWS` do not resemble Adobe Flash.


[^1]: I do confess, I let ChatGPT have a crack at it and it gave me some comically wrong answers, including: 1. making up the command `cvmfs_find <repo> --catalogs`, which it claims is a "helper tool to search metadata, including nested catalogs", 2. making up the `cvmfs_server catalog-chroot` subcommand "to mount the catalog structure", and 3. making up the `cvmfs_swissknife cat` subcommand to extract catalogs. Oh you wacky LLM and your confidently incorrect assertions. You'd make a great redditor.

[gtdb]: https://gtdb.ecogenomic.org/
[cvmfs]: https://cvmfs.readthedocs.io/en/stable/
