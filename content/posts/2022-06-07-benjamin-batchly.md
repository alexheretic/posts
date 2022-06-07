---
title: "Rust async batching with benjamin-batchly"
---
Sometimes instead of doing lots of little things concurrently, it is better to bundle them together
and do them all at once, as a _batch_. So on a bank holiday Thursday morning I awoke early (mostly due to the screams of my 1 year old boys) and (after the screaming stopped) wrote a crate to help do that: [benjamin_batchly](https://github.com/alexheretic/benjamin-batchly).

## Example: Inserting into a database
A recurring theme of database optimisation is reducing "trips" to the database. It is usually faster to send 1 fat message containing N items than it is to send N thin messages containing just 1 item.

Consider a crud-style create request which we'd like to insert into a database.

```rust
async fn handle_create_foo(db: &Db, data: CreateFooRequest) -> Result<(), Error> {
    let db_item = DbItem::from(data);
    db.insert(db_item).await?;
    Ok(())
}
```
So for each _CreateFooRequest_ we will insert a single item into our db. We can batch these with a _BatchMutex_.
```rust
use benjamin_batchly::{BatchMutex, BatchResult};

async fn handle_create_foo(
    batch_mutex: &BatchMutex<(), DbItem>,
    db: &Db,
    data: CreateFooRequest,
) -> Result<(), Error> {
    let db_item = DbItem::from(data);
    match batch_mutex.submit((), db_item).await {
        BatchResult::Work(mut batch) => {
            db.bulk_insert(&batch.items).await?;
            batch.notify_all_done();
            Ok(())
        }
        BatchResult::Done(_) => Ok(()),
        BatchResult::Failed => Err(Error::BatchFailed),
    }
}
```
Submitting to the _BatchMutex_ provides 3 async outcomes. 
* ***Work*** A batch of 1 or more items, including the one submitted, which should be handled.
* ***Done*** The submitted item was handled in a batch by another submitter & notified done.
* ***Failed*** The submitted item was handled in a batch by another submitter but dropped before notifying done.

So if 100 _CreateFooRequest_ requests come in while a batch is ongoing they'll await at _submit_. Once the previous batch has finished all waiting submissions become the next batch. 1 call will return _BatchResult::Work_ and after the _batch.notify_all_done()_ call 99 others will return _BatchResult::Done_.

Note: In the example I used the unit type () as the first argument for _submit_. This is the "batch key" which can be used to partition batching. Only items submitted with the same batch key are bundled together which is useful in more general cases. 

Note(2): Individual item return values are also supported, check out the [docs](https://docs.rs/benjamin_batchly/latest/benjamin_batchly) for more info.

## benjamin-batchly implementation goals
Some I had goals for the implementation:
* Avoid spawning any additional tasks.
* Avoid lifetime issues (so probably avoid async closures).

By using a async return value instead of a closure it's clear who is doing the work here and avoids lifetime borrowing & move issues in the abstraction.

* All submissions **must** know whether the batch worked or not.
* Usage should be as "foolproof" as possible.

One of the wonderful Rust concepts that really allows this lib to shine is the [Drop trait](https://doc.rust-lang.org/std/ops/trait.Drop.html). It eliminates foot guns by reliably providing feedback to submit callers in the edge cases. Love it.

* Batching should have minimal latency overhead.

In my motivating use case I'm fine with batch sizes of 1 for low traffic and I didn't want to wait artificially. The currently implementation the batch size is totally driven by how long it takes to do the work and how many new submissions are incoming. So slow processes with high traffic will naturally see bigger batches. Submissions are never waiting around for a batch, if they can go, they go. Maybe in future the crate should also handing some form of configurable waiting, if there's a good use case?


