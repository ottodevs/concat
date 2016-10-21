# Concat

_concat<sup>[1](#footnote-1)</sup>: concatenating cat, by contrast to [copycat](https://github.com/atomix/copycat), used in a [previous prototype](https://github.com/mediachain/oldchain)_

**concat** is a set of daemons that provide the backbone of the Mediachain peer-to-peer network.
Please see [RFC 4](https://github.com/mediachain/mediachain/blob/master/rfc/mediachain-rfc-4.md), [concat.md](https://github.com/mediachain/mediachain/blob/master/docs/concat.md) and the [9/15/16](https://blog.mediachain.io/looking-backwards-looking-forwards-9149bf00f876#.c4xrhcdwj) developer update for a high level overview of this design.

The two main programs in concat are **mcnode**, which is the implementation of a fully featured Mediachain node, and **mcdir**, which implements a directory server for facilitating node discovery and connectivity.

## Installation
### Precompiled Binaries

You can download the latest release `mcnode` binary for your platform (Linux or Mac) from [releases](https://github.com/mediachain/concat/releases).

### Installing from Source
Concat requires Go 1.7 or later.

First clone the repo in your `$GOPATH`:
```
$ mkdir -p $GOPATH/src/github.com/mediachain
$ git clone https://github.com/mediachain/concat.git $GOPATH/src/github.com/mediachain/concat
```

You can then run the setup and install scripts which will build dependencies and install the programs in `$GOPATH/bin`:
```
$ cd $GOPATH/src/github.com/mediachain/concat
$ ./setup.sh && ./install.sh 
```

Warning: `setup.sh` is quite slow; among the dependencies is gorocksdb, which takes several minutes to compile.

## Usage

### Getting Started
You can run your local mediachain node by invoking `mcnode` without arguments:
```
$ mcnode
2016/10/20 19:29:24 Generating new node identity
2016/10/20 19:29:25 Saving key to /home/vyzo/.mediachain/mcnode/identity.node
2016/10/20 19:29:25 Node ID: QmeBkfxcaBfA9pvzivRwhF2PM7sXpp4HHQbp7jfTkRCWEa
2016/10/20 19:29:25 Generating new publisher identity
2016/10/20 19:29:25 Saving key to /home/vyzo/.mediachain/mcnode/identity.publisher
2016/10/20 19:29:25 Publisher ID: 4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey
2016/10/20 19:29:25 Node is offline
2016/10/20 19:29:25 Serving client interface at 127.0.0.1:9002
```

The first time you run `mcnode`, it will generate a pair of persistent identities and
initialize the local store. By default, `mcnode` uses `~/.mediachain/mcnode` as its
root directory, but you can change this using the `-d path/to/mcnode/home` command line
option.

`mcnode` is intended to be run as a daemon, so you can run it in a docker container,
use `daemon` to daemonize, or simply run it inside a `screen`.

By default, the node starts disconnected from the p2p network and provides a client control
api in localhost.
You can control and use your node through the api using curl, but it is recommended
to install [Aleph](https://github.com/mediachain/aleph).
Aleph provides a fully featured client for `mcnode` named `mcclient` and we use it in
the examples below.

Once your node is up and running you can check its status:
```
$ mcclient id
Peer ID: QmeBkfxcaBfA9pvzivRwhF2PM7sXpp4HHQbp7jfTkRCWEa
Publisher ID: 4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey
Info: 
$ mcclient status
offline
```

At this point you can you should designate a directory for looking up peers:
```
$ mcclient config dir /ip4/54.173.71.205/tcp/9000/Qmb561BUvwc2pfMQ5NHb1uuwVE1hHWerySVMSUJLhnzNLN
set directory to /ip4/54.173.71.205/tcp/9000/Qmb561BUvwc2pfMQ5NHb1uuwVE1hHWerySVMSUJLhnzNLN
```

And your node is ready to go online:
```
$ mcclient status online
status set to online
$ mcclient listPeers
...
QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ
...
$ mcclient id QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ
Peer ID: QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ
Publisher ID: 4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm
Info: Metadata for CC images from DPLA, 500px, and pexels; operated by Mediachain Labs.
```

### Basic Operations

The basic mediachain operations are queries, local or remote,  and merges of remote datasets.

Here, we query the namespaces for statements in the discovered peer's datastore and take a small
sample from the `images.dpla` namespace:
```
$ mcclient query -r QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ "SELECT namespace FROM *"
'images.500px'
'images.dpla'
'images.pexels'

$ mcclient query -r QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ "SELECT * FROM images.dpla LIMIT 5"
{ id: '4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm:1476970964:0',
  publisher: '4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm',
  namespace: 'images.dpla',
  body: 
   { Body: 
      { Simple: 
         { object: 'Qma1LUdw5PAjfuZLXCbT5Qm5xnQFLkEejyXbLuvcKinF8K',
           refs: [ 'dpla_1349ede833fa9417b6f55be6bb402f6d' ] } } },
  timestamp: 1476970964,
  signature: 'mZMSMdMrahd40uyJChHAFLMvB5diR8qh9QI2kw0XUR7HxTo6sh2jtCzHZVBaKnOa7w9QkSrPdEU8qqfAsBEiDA==' }
...  
```

We can merge remote datasets using a query; the node will merge in
the statements selected by the query and associated metadata:

```
$ mcclient merge QmeiY2eHMwK92Zt6X4kUUC3MsjMmVb2VnGZ17DhnhRPCEQ "SELECT * FROM images.dpla LIMIT 5"
merged 5 statements and 5 objects
```

The statements and the associated metadata our now stored in the local node:
```
$ mcclient query "SELECT * FROM images.dpla"
{ id: '4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm:1476970964:0',
  publisher: '4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm',
  namespace: 'images.dpla',
  body: 
   { Body: 
      { Simple: 
         { object: 'Qma1LUdw5PAjfuZLXCbT5Qm5xnQFLkEejyXbLuvcKinF8K',
           refs: [ 'dpla_1349ede833fa9417b6f55be6bb402f6d' ] } } },
  timestamp: 1476970964,
  signature: 'mZMSMdMrahd40uyJChHAFLMvB5diR8qh9QI2kw0XUR7HxTo6sh2jtCzHZVBaKnOa7w9QkSrPdEU8qqfAsBEiDA==' }
...

$ mcclient getData Qma1LUdw5PAjfuZLXCbT5Qm5xnQFLkEejyXbLuvcKinF8K
{ orientation: null,
  dedupe_hsh: '4e0783f1e8f41bac',
  licenses: [ { details: null } ],
  native_id: 'dpla_http://dp.la/api/items/1349ede833fa9417b6f55be6bb402f6d',
  keywords: [],
  date_created_original: null,
  title: [ 'Page 91' ],
  camera_exif: {},
  source: { url: 'https://dp.la/', name: 'dpla' },
  transient_info: { score_hotnessviews: null, likes: null },
  date_captured: null,
  location: { lat_lon: null, place_name: [] },
  attribution: null,
  description: 'kada',
  source_tags: [ 'dp.la' ],
  date_created_at_source: null,
  providers_list: 
   [ { name: 'dpla' },
     { name: 'University of Southern California. Libraries' } ],
  date_source_version: null,
  sizes: 
   [ { bytes: null,
       height: 120,
       uri_external: 'http://digitallibrary.usc.edu/utils/getthumbnail/collection/p15799coll126/id/7763',
       width: 66,
       content_type: 'image/jpeg',
       dpi: null } ],
  source_dataset: 'dpla',
  artist_names: [],
  url_direct: { url: 'http://digitallibrary.usc.edu/utils/getthumbnail/collection/p15799coll126/id/7763' },
  derived_qualities: 
   { medium: null,
     predicted_tags: null,
     has_people: null,
     colors: null,
     general_type: null,
     time_period: null },
  url_shown_at: { url: 'http://digitallibrary.usc.edu/cdm/ref/collection/p15799coll126/id/7763' },
  aspect_ratio: 0.55 }

```

If you decide you no longer want to keep the statements you merged in your local store,
you can delete them:
```
$ mcclient delete "DELETE FROM images.dpla"
Deleted 5 statements
```

### Publishing Statements

You can publish statements to your local node by creating json objects with the metadata
you want to publish.

For example, let's publish hello world in a couple different variations:
```
$ cat /tmp/hello.json 
{"id": "hello_1", "hello": "world"}
{"id": "hello_2", "hola": "mundo"}
$ mcclient publish --idSelector 'id' scratch.hello /tmp/hello.json
statement id: 4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey:1477063161:0 -- body: QmZDxgNgUT1J3rgjvnGjoxoA5efGNSN9Qvhq4FpvefmwnA -- refs: ["hello_1"]
statement id: 4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey:1477063161:1 -- body: QmcdACgobENfs7vuD2CpQGi8RdEkNyAmbmyqVvrA7Z57xu -- refs: ["hello_2"]
All statements published successfully
```

The statements and the metadata are now stored by the local node:
```
$ mcclient query 'SELECT * FROM scratch.hello'
{ id: '4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey:1477063161:0',
  publisher: '4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey',
  namespace: 'scratch.hello',
  body: 
   { Body: 
      { Simple: 
         { object: 'QmZDxgNgUT1J3rgjvnGjoxoA5efGNSN9Qvhq4FpvefmwnA',
           refs: [ 'hello_1' ] } } },
  timestamp: 1477063161,
  signature: 'Q79Vgp7bl4J7rJo3DOmtueLGqHQpG3lpSyiXbTqX60oOdbPdju06m4epiQNrGLarfCsj2Opa0psX6EWm6BJkDw==' }
{ id: '4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey:1477063161:1',
  publisher: '4XTTMADSKQUN3jkeZngbtuE35w9y5YnDTicVTeeji7N2Npkey',
  namespace: 'scratch.hello',
  body: 
   { Body: 
      { Simple: 
         { object: 'QmcdACgobENfs7vuD2CpQGi8RdEkNyAmbmyqVvrA7Z57xu',
           refs: [ 'hello_2' ] } } },
  timestamp: 1477063161,
  signature: 'bN7mAW3JKJAHtInSGM08IXoUx//qFJpySaCWPEe8P6Pm46XxoOrepG2Q2KrIQ5bs5oLKJkU7qHqFI2dQaHETCQ==' }

$ mcclient getData QmZDxgNgUT1J3rgjvnGjoxoA5efGNSN9Qvhq4FpvefmwnA
{ id: 'hello_1', hello: 'world' }
```

### Going Public

In order to distribute statements stored in your node, it needs to become
discoverable and accessible to other nodes. This happens by taking the node
`public`, which registers with the directory.

Before you can take your node public however, you need to ensure that it is
reachable in the network. By default, `mcnode` binds its p2p interface in port 9001
for all interfaces, so if your node is directly connected to the Internet (eg in a vps
host), you don't have to do anything.

However, chances are your node is behind a NAT and you will need to
configure it for traversal through the `nat` option.

If you are behind a home router which supports upnp port mapping, you can
set your NAT configuration to "auto":
```
$ mcclient config nat auto
```

If you are behind multiple NATs or behind a firewall that doesn't support port mapping
(eg an AWS instance), then you need to manually configure your network to route traffic
to the p2p port in your host.
If you mapped the port transparently, you can set your NAT configuration to "*", which
will automatically detect the IP address and advertise the locally bound port:
```
$ mcclient config nat "*"
```

If you have mapped a different port to your host interface, then you need to specify
it explicitly:
```
$ mcclient config nat "*:port"
```

Once you've configured your node's NAT traversal, you should add a short description
about your node and its contents:
```
$ mcclient config info "Metadata from ...; Operated by ..."
```

At this point, your node should be ready and reachable in the network, and you can take it public:
```
$ mcclient status public
```
This will register the node with the configured directory, which will make it visible
to other nodes through `mcclient listPeers`.

If you want to take your node back offline, you can simply do so with `status offline`:
```
$ mcclient status offline
```

## mcnode
### Architecture
The node contains the **statement db** and the **datastore**.

The datastore contains the metadata _per se_, as CBOR objects ([IPLD](https://github.com/ipld/specs/tree/master/ipld) compatible to the best of our ability) of unspecified schema, stored in RocksDB in point lookup mode.

The statement db contains **statements** about one (currently) or more metadata objects: their publisher, namespace, timestamp and signature. Statements are [protobuf objects](https://github.com/mediachain/concat/blob/master/proto/stmt.proto) sent over the wire between peers to signal publication or sharing of metadata; when stored, they act as an index to the datastore. This db is currently stored in SQLite.

### MCQL
MCQL is a query language for retrieving statements from the node's statement db.
It supports `SELECT` (and `DELETE`) statements with a syntax very similar to SQL, where
namespaces play the role of tables.

Some basic MCQL statements:

```sql
-- see all namespaces in the database
SELECT namespace FROM *

-- count all statements in the namespace images.dpla
SELECT COUNT(*) FROM images.dpla

-- retrieve the last 5 statements by timestamp
SELECT * FROM images.dpla ORDER BY timestamp DESC LIMIT 5

-- retrieve the last 5 statements by insertion order in the db
SELECT * FROM images.dpla ORDER BY counter DESC LIMIT 5

-- retrieve statement id, insertion counter tuples
SELECT (id, counter) FROM images.dpla

-- see all publishers in the namespace
SELECT publisher FROM images.dpla

-- retrieve all statements by a publisher
SELECT * FROM * WHERE publisher = 4XTTM4K8sqTb7xYviJJcRDJ5W6TpQxMoJ7GtBstTALgh5wzGm

```

The full grammar for MCQL is defined as a PEG in [query.peg](mc/query/query.peg)

### REST API
A REST API is provided for controlling the node. This is an administrative interface and should NOT be accessible to the wider network.

* `GET /id` -- node info for this peer
* `GET /id/{peerId}` -- node info for peer given by peerId
* `GET /ping/{peerId}` -- ping!
* `POST /publish/{namespace}` -- publish a batch of statements to the specified namespace 
* `GET /stmt/{statementId}` -- retrieve statement by statementId
* `POST /query` -- issue MCQL SELECT query on this peer
* `POST /query/{peerId}` -- issue MCQL SELECT query on a remote peer
* `POST /merge/{peerId}` -- query a remote peer and merge the resulting statements into this one
* `POST /delete` -- delete statements matching this MCQL DELETE query
* `POST /data/put` -- add a batch of data objects to datastore
* `GET /data/get/{objectId}` -- get an object from the datastore
* `GET /status` -- get node network state
* `POST /status/{state}` -- control network state (online/offline/public)
* `GET/POST /config/dir` -- retrieve/set the configured directory
* `GET/POST /config/nat` -- retrieve/set NAT setting
* `GET/POST /config/info` -- retrieve/set info string
* `GET /dir/list` -- list known peers
* `GET /net/addr` -- list known addresses

### P2P API

TODO

## mcdir
See also [roles](https://github.com/mediachain/mediachain/blob/master/rfc/mediachain-rfc-4-roles.md#directory-servers).

### P2P API

TODO


1. <a name="footnote-1"></a> alternately [funes](http://www4.ncsu.edu/~jjsakon/FunestheMemorious.pdf), cf. [aleph](https://github.com/mediachain/aleph)
