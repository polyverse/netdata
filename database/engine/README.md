# Database engine

The Database Engine works like a traditional
database. There is some amount of RAM dedicated to data caching and indexing and the rest of
the data reside compressed on disk. The number of history entries is not fixed in this case,
but depends on the configured disk space and the effective compression ratio of the data stored.

## Files

With the DB engine memory mode the metric data are stored in database files. These files are
organized in pairs, the datafiles and their corresponding journalfiles, e.g.:

```
datafile-1-0000000001.ndf
journalfile-1-0000000001.njf
datafile-1-0000000002.ndf
journalfile-1-0000000002.njf
datafile-1-0000000003.ndf
journalfile-1-0000000003.njf
...
``` 

They are located under their host's cache directory in the directory `./dbengine`
(e.g. for localhost the default location is `/var/cache/netdata/dbengine/*`). The higher
numbered filenames contain more recent metric data. The user can safely delete some pairs
of files when netdata is stopped to manually free up some space.

*Users should* **back up** *their `./dbengine` folders if they consider this data to be important.*

## Configuration

There is one DB engine instance per netdata host/node. That is, there is one `./dbengine` folder
per node, and all charts of `dbengine` memory mode in such a host share the same storage space
and DB engine instance memory state. You can select the memory mode for localhost by editing
netdata.conf and setting:

```
[global]
    memory mode = dbengine
```

For setting the memory mode for the rest of the nodes you should look at
[streaming](../../streaming/).

The `history` configuration option is meaningless for `memory mode = dbengine` and is ignored
for any metrics being stored in the DB engine.

All DB engine instances, for localhost and all other streaming recipient nodes inherit their
configuration from `netdata.conf`:

```
[global]
    page cache size = 32
    dbengine disk space = 256
```

The above values are the default and minimum values for Page Cache size and DB engine disk space
quota. Both numbers are in **MiB**. All DB engine instances will allocate the configured resources
separately.

The `page cache size` option determines the amount of RAM in **MiB** that is dedicated to caching
netdata metric values themselves.

The `dbengine disk space` option determines the amount of disk space in **MiB** that is dedicated
to storing netdata metric values and all related metadata describing them.

## Operation

The DB engine stores chart metric values in 4096-byte pages in memory. Each chart dimension gets
its own page to store consecutive values generated from the data collectors. Those pages comprise
the **Page Cache**.

When those pages fill up they are slowly compressed and flushed to disk.
It can take `4096 / 4 = 1024 seconds = 17 minutes`, for a chart dimension that is being collected
every 1 second, to fill a page. Pages can be cut short when we stop netdata or the DB engine
instance so as to not lose the data. When we query the DB engine for data we trigger disk read
I/O requests that fill the Page Cache with the requested pages and potentially evict cold
(not recently used) pages. 

When the disk quota is exceeded the oldest values are removed from the DB engine at real time, by
automatically deleting the oldest datafile and journalfile pair. Any corresponding pages residing
in the Page Cache will also be invalidated and removed. The DB engine logic will try to maintain
between 10 and 20 file pairs at any point in time. 

The Database Engine uses direct I/O to avoid polluting the OS filesystem caches and does not 
generate excessive I/O traffic so as to create the minimum possible interference with other 
applications.

## Memory requirements

Using memory mode `dbengine` we can overcome most memory restrictions and store a dataset that
is much larger than the available memory.

There are explicit memory requirements **per** DB engine **instance**, meaning **per** netdata 
**node** (e.g. localhost and streaming recipient nodes):

- `page cache size` must be at least `#dimensions-being-collected x 4096 x 2` bytes.

- an additional `#pages-on-disk x 4096 x 0.03` bytes of RAM are allocated for metadata.

    - roughly speaking this is 3% of the uncompressed disk space taken by the DB files.

    - for very highly compressible data (compression ratio > 90%) this RAM overhead
      is comparable to the disk space footprint.

An important observation is that RAM usage depends on both the `page cache size` and the 
`dbengine disk space` options. 

[![analytics](https://www.google-analytics.com/collect?v=1&aip=1&t=pageview&_s=1&ds=github&dr=https%3A%2F%2Fgithub.com%2Fnetdata%2Fnetdata&dl=https%3A%2F%2Fmy-netdata.io%2Fgithub%2Fdatabase%2Fengine%2FREADME&_u=MAC~&cid=5792dfd7-8dc4-476b-af31-da2fdb9f93d2&tid=UA-64295674-3)]()
