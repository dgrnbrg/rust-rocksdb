rust-rocksdb
============
[![Build Status](https://travis-ci.org/spacejam/rust-rocksdb.svg?branch=master)](https://travis-ci.org/spacejam/rust-rocksdb)

This library has been tested against RocksDB 3.8.1 on linux and OSX.  The 0.0.7 crate should work with the Rust nightly release as of 7/16/15.

### status
  - [x] basic open/put/get/delete/close
  - [x] rustic merge operator
  - [x] write batch (thanks @dgrnbrg!)
  - [x] compaction filter, style
  - [x] LRU cache
  - [x] destroy/repair
  - [x] iterator
  - [x] comparator
  - [x] snapshot
  - [ ] column family operations
  - [ ] slicetransform
  - [ ] windows support

Feedback and pull requests welcome!  If a particular feature of RocksDB is important to you, please let me know by opening an issue, and I'll prioritize it.

###### Prerequisite: RocksDB
```bash
wget https://github.com/facebook/rocksdb/archive/rocksdb-3.8.tar.gz
tar xvf rocksdb-3.8.tar.gz && cd rocksdb-rocksdb-3.8 && make shared_lib
sudo make install
```

### Running
###### Cargo.toml
```rust
[dependencies]
rocksdb = "~0.0.7"
```
###### Code
```rust
extern crate rocksdb;
use rocksdb::{RocksDB, Writable};

fn main() {
    let mut db = RocksDB::open_default("/path/for/rocksdb/storage").unwrap();
    db.put(b"my key", b"my value");
    db.get(b"my key")
        .map( |value| { 
            println!("retrieved value {}", value.to_utf8().unwrap())
        })
        .on_absent( || { println!("value not found") })
        .on_error( |e| { println!("operational problem encountered: {}", e) });

    db.delete(b"my key");
}
```

###### Doing an atomic commit of several writes
```rust
extern crate rocksdb;
use rocksdb::{RocksDB, WriteBatch, Writable};

fn main() {
    // NB: db is automatically freed at end of lifetime
    let mut db = RocksDB::open_default("/path/for/rocksdb/storage").unwrap();
    {
        let mut batch = WriteBatch::new(); // WriteBatch and db both have trait Writable
        batch.put(b"my key", b"my value");
        batch.put(b"key2", b"value2");
        batch.put(b"key3", b"value3");
        db.write(batch); // Atomically commits the batch
    }
}
```

###### Getting an Iterator
```rust
extern crate rocksdb;
use rocksdb::{RocksDB, Direction};

fn main() {
    // NB: db is automatically freed at end of lifetime
    let mut db = RocksDB::open_default("/path/for/rocksdb/storage").unwrap();
    let mut iter = db.iterator();
    for (key, value) in iter.from_start() { // Always iterates forward
        println!("Saw {} {}", key, value); //actually, need to convert [u8] keys into Strings
    }
    for (key, value) in iter.from_end() { //Always iterates backward
        println!("Saw {} {}", key, value);
    }
    for (key, value) in iter.from(b"my key", Direction::forward) { // From a key in Direction::{forward,reverse}
        println!("Saw {} {}", key, value);
    }
}
```

###### Getting an Iterator
```rust
extern crate rocksdb;
use rocksdb::{RocksDB, Direction};

fn main() {
    // NB: db is automatically freed at end of lifetime
    let mut db = RocksDB::open_default("/path/for/rocksdb/storage").unwrap();
    let snapshot = db.snapshot(); // Creates a longer-term snapshot of the DB, but freed when goes out of scope
    let mut iter = snapshot.iterator(); // Make as many iterators as you'd like from one snapshot
}
```

###### Rustic Merge Operator
```rust
extern crate rocksdb;
use rocksdb::{Options, RocksDB, MergeOperands, Writable};

fn concat_merge(new_key: &[u8], existing_val: Option<&[u8]>,
    operands: &mut MergeOperands) -> Vec<u8> {
    let mut result: Vec<u8> = Vec::with_capacity(operands.size_hint().0);
    existing_val.map(|v| { result.extend(v) });
    for op in operands {
        result.extend(op);
    }
    result
}

fn main() {
    let path = "/path/to/rocksdb";
    let mut opts = Options::new();
    opts.create_if_missing(true);
    opts.add_merge_operator("test operator", concat_merge);
    let mut db = RocksDB::open(&opts, path).unwrap();
    let p = db.put(b"k1", b"a");
    db.merge(b"k1", b"b");
    db.merge(b"k1", b"c");
    db.merge(b"k1", b"d");
    db.merge(b"k1", b"efg");
    let r = db.get(b"k1");
    assert!(r.unwrap().to_utf8().unwrap() == "abcdefg");
}
```

###### Apply Some Tunings
Please read [the official tuning guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide), and most importantly, measure performance under realistic workloads with realistic hardware.
```rust
use rocksdb::{Options, RocksDB};
use rocksdb::RocksDBCompactionStyle::RocksDBUniversalCompaction;

fn badly_tuned_for_somebody_elses_disk() -> RocksDB {
    let path = "_rust_rocksdb_optimizetest";
    let mut opts = Options::new();
    opts.create_if_missing(true);
    opts.set_max_open_files(10000);
    opts.set_use_fsync(false);
    opts.set_bytes_per_sync(8388608);
    opts.set_disable_data_sync(false);
    opts.set_block_cache_size_mb(1024);
    opts.set_table_cache_num_shard_bits(6);
    opts.set_max_write_buffer_number(32);
    opts.set_write_buffer_size(536870912);
    opts.set_target_file_size_base(1073741824);
    opts.set_min_write_buffer_number_to_merge(4);
    opts.set_level_zero_stop_writes_trigger(2000);
    opts.set_level_zero_slowdown_writes_trigger(0);
    opts.set_compaction_style(RocksDBUniversalCompaction);
    opts.set_max_background_compactions(4);
    opts.set_max_background_flushes(4);
    opts.set_filter_deletes(false);
    opts.set_disable_auto_compactions(true);

    RocksDB::open(&opts, path).unwrap()
}
```

