---
layout: post
title: "How We Cut a Service's Memory Budget by 90% and Made It 5x Faster"
date: 2026-08-25
---

### A single BLOB, a single listener, and a heap graph that climbed with every upload — until we made the chunk the unit of everything.

---

> **TL;DR**
> - **Problem:** 1 GB CSVs stored as a single BLOB, parsed whole by one listener. OOM kills, hour-long hangs, no way to scale out.
> - **Fix:** Make the chunk — not the file — the unit of storage, messaging, and processing.
> - **Result:** 2 GB → 200 MB memory. The same 500 MB file: 43 min → 8 min. The 1 GB file that used to OOM: 13 min. Zero crashes since.
> - **The interesting part:** durable row-level progress turns an at-least-once queue into effectively-once processing, and hands you completion detection and output ordering for free.

---

Every enterprise product eventually grows the same feature: a screen where users upload a CSV with hundreds of thousands — sometimes millions — of rows, and another where they download one.

Ours had it too. It worked fine in demos. It worked fine in testing. Then real customers showed up with real files — uploads pushing **a gigabyte, a million rows** — and our service started doing its best impression of a memory leak. The module had 2 GB of RAM allocated. A 500 MB file took **43 minutes** and spiked memory past 90%. A 1 GB file didn't finish at all: Kubernetes OOM-killed the pod mid-parse, taking not just the job but the whole service down with it. Under lighter load it wouldn't crash — it would just hang, one listener grinding away for over an hour on a single thread.

Today that same module runs on **200 MB** of RAM — a tenth of what it had — and puts that same 500 MB file through in **8 minutes**. The gigabyte file that never used to finish now takes **13**. It hasn't OOM-killed once since the rollout, and has never even tripped the 90% memory alert.

As the lead on this area, the redesign landed on my desk. This is the story of how the fix wasn't a bigger heap or fancier storage, but a boring old idea: **never hold the whole file anywhere, ever.**

---

## The old design, and why it was slowly killing us

The original pipeline was simple — which is why it was easy to ship and painful to scale:

{% include diagram-before.svg %}

Four things multiplied against us:

- **The ORM loads LOBs whole.** The file lived in a single row, so every read or write materialized the *entire* content on the heap. There is no "peeking at" a 1 GB BLOB through an ORM entity — you get the whole gigabyte, every time anything touches that row.
- **Parsing multiplied it.** A 1 GB CSV becomes several gigabytes once parsed into an in-memory list of objects — object headers, wrapper types, a string per cell. One job could blow past the pod's 2 GB limit, and Kubernetes doesn't negotiate: it OOM-kills the container, mid-job.
- **All-or-nothing.** The file was processed in one shot. An OOM kill at row 900,000? Start over. No partial progress, no resumption.
- **One listener per job — and we couldn't add more.** Big files simply outlasted every timeout around them, but scaling out wasn't an option: more listeners would mean multiple instances tearing through the same large files in parallel with no strong consistency guarantees. The single listener wasn't an oversight; it was the only *safe* choice the design left us. So our one escape hatch — horizontal scaling — was welded shut.

None of these were bugs. It was a design that assumed files were small. Files stopped being small.

---

## The one-sentence fix

After enough heap dumps and stuck jobs, the whole redesign collapsed into a single decision:

> **Split the file into chunks at upload time — and make the chunk, not the file, the unit of storage, messaging, and processing.**

Once the chunk is the unit of work, memory stops scaling with file size and becomes proportional to *chunk* size — a constant we control. Parallelism, retries, and resumability all fell out of that one decision, mostly for free.

I designed the framework around that principle. My team and I built it out together. Here's how it works.

---

## Write path: chunk on the way in

The upload is never buffered. We read the incoming stream line by line, and every time we accumulate one batch (say, 1,000 rows), we flush it as **one small row** in a chunk table and reset the buffer:

```
# pseudocode
for each line in the incoming stream:
    append line to buffer
    if buffer holds BATCH_SIZE rows:
        save buffer as chunk (fileId, next sequence number)
        clear buffer

save whatever is left in the buffer as the final chunk
```

A "file" is now just a UUID plus an ordered set of chunk rows. Two design calls did quiet heavy lifting here:

- **Each chunk commits in its own transaction.** A 1,000-chunk upload is 1,000 fast commits, not one monster transaction holding locks for minutes.
- **The header row never enters a chunk.** It's peeled off and stored in job metadata, so every chunk has exactly the same shape — which is what makes the next part possible.

---

## Processing: one message per chunk, not per job

This is where the "single listener grinds through the whole job" model got dismantled. When a job runs, the orchestrator publishes **one queue message per chunk**. Each message is self-describing: job ID, chunk sequence, batch size, column headers.

{% include diagram-after.svg %}

A worker fetches *only its chunk*, streams it row by row into entities (never materializing even one full chunk as objects), processes it, and reports back. Three things fall out of that:

- **Safe parallelism, finally.** Workers can chew on many chunks of the same file simultaneously, because each chunk is a predictable-sized, independent unit with its own message. Horizontal scaling went from forbidden to being the whole point — and crucially, this isn't per-pod concurrency. Any pod can pick up any chunk of any file. Scaling the service scales the throughput.
- **No more timeouts.** A unit of work is now 1,000 rows — seconds, not hours.
- **Failure shrinks.** A worker dies mid-chunk? One message is redelivered, one batch reprocessed. The other 999 chunks don't care.

One subtlety we had to get right: **row numbering.** The first error reports we produced numbered every chunk from zero — technically correct, completely useless to a user staring at row 481,203 of their spreadsheet. Since every chunk holds exactly one batch, the fix is pure arithmetic: `firstRow = chunkSequence × batchSize`. Tiny detail; it's the difference between an error report users can act on and one they can't.

---

## "One message is redelivered" — so how do you avoid processing rows twice?

This is the question the design lives or dies on, and it deserves a section of its own. Our queue gives us *at-least-once* delivery. A worker can crash after processing 700 rows of a 1,000-row chunk, and that message will come back. Without care, you get 700 duplicate rows.

The answer is a **staging table** that makes progress durable at row granularity. It's deliberately dumb — five columns:

```
row_text      the exact CSV row, verbatim
status        PENDING / PROCESSED / FAILED
failure_reason
chunk_id
job_id
```

A worker's first act is not to process anything. It **explodes its chunk into staging rows** and commits them as `PENDING`. Only then does it start work, flipping each row's status as it goes.

That ordering is what makes replay safe:

```
# pseudocode — every worker starts here
if staging rows already exist for (job_id, chunk_id):
    skip the fetch and the explode      # someone got here before us
else:
    fetch chunk, explode into staging rows as PENDING

process only the rows still PENDING     # already-PROCESSED rows are skipped
```

A redelivered message finds most of its work already done and picks up exactly where the dead worker stopped. At-least-once delivery becomes effectively-once processing, without distributed locks or an exactly-once queue we don't have.

The staging table pays a second dividend: **completion detection needs no coordinator.** Every worker, on finishing its chunk, runs one count:

```sql
SELECT count(*) FROM staging WHERE job_id = ? AND status <> 'PENDING'
```

If that equals the file's total row count, the last worker to finish — whichever one it happens to be — marks the job complete. No orchestrator tracking outstanding chunks, no timeout waiting for stragglers. The workers figure it out among themselves.

And a third: **ordering is preserved on the way out.** Results are streamed back to the file store in staging-row order, so the output CSV matches the input CSV line for line, even though the rows were processed by different workers on different pods in whatever order the queue happened to deliver them. Users get their errors on the rows they recognize.

Once a job's results are written out, the staging rows have no further purpose. A scheduled purge policy clears them.

---

## Read path: stream out, one chunk at a time

Downloads got the same treatment in reverse. We query for the chunk *IDs* — not the chunks — then load one at a time, write it to the response stream, flush, and let it become garbage before touching the next:

```
# pseudocode
chunkIds = fetch chunk IDs for the file, in order    # IDs only — cheap

for each chunkId:
    load that one chunk                              # one chunk in memory, ever
    write it to the response stream
    flush                                            # gone before the next arrives
```

The client sees a normal file download. The server's heap sees a ripple instead of a wave:

{% include diagram-heap.svg %}

---

## "Why didn't you just use object storage?"

Fair question — files-in-the-database is nobody's dream architecture. The honest answer: **the production environments we deployed into couldn't give us one.** No object store, no blessed shared volume. The database was the only durable, replicated, backed-up storage available everywhere the product ran.

That constraint was real, but choosing the database is not free, and I'd rather name the bill than pretend we didn't pay it:

- **We roughly double the bytes on disk, transiently.** A 1 GB upload lands as ~1 GB of chunk rows, and then the staging table explodes it into another ~1 GB of row text while the job runs — plus indexes on both. Object storage would have kept all of that out of the database entirely.
- **Every one of those bytes hits the replication stream.** WAL, replicas, backups. We inflated backup size and replication traffic with data that is transient by design and worthless twelve hours later. This, not disk space, is the cost that actually hurts.
- **The purge policy is load-bearing, not housekeeping.** Deleting millions of staging rows produces dead tuples and IO churn of its own. Purge windows have to be tuned so they don't collide with peak upload hours — a maintenance concern that simply doesn't exist if the bytes live in a bucket with a lifecycle rule.
- **Bulk writes compete with transactional traffic.** The chunk and staging writes share a connection pool and IOPS budget with the service's normal OLTP work. Above a certain concurrency, file processing and the rest of the product start taxing each other.
- **Per-gigabyte, this is expensive storage.** Replicated, backed-up database storage on premium disk costs multiples of what object storage costs for the same bytes.

We accepted every one of these because the alternative was a service that fell over on real customer files. But it's a trade, not a win.

Which is exactly why I insisted the storage logic live behind an interface. Everything above it speaks only in file references, chunk sequences, and streams — nothing knows the chunks live in a database table. The day an environment offers MinIO or S3, we swap the implementation and delete most of the costs above without touching a line of the framework.

The real architecture was never *where* the bytes live. It's the contract that **no caller may ever ask for the whole file at once.** That holds no matter what's behind the interface.

---

## Migrating without a big bang

We couldn't flip dozens of existing job types onto a new framework overnight — and I wasn't going to ask the team to try. So each job *definition* carries a framework version. Legacy job types keep flowing through the old path; migrated ones get the chunked fan-out. Both share the same job table, status lifecycle, and storage layer, so migration is a per-job-type config change. The team migrates a few types each release, and the deprecated path shrinks every time.

---

## What actually changed

| | Before | After |
|---|---|---|
| Memory allocated to the module | 2 GB | **200 MB** |
| Peak memory under load | >90%, routine OOM kills | never tripped the 90% alert |
| 500 MB file | 43 minutes | **8 minutes** |
| 1 GB file, 1M rows | never finished — OOM | **13 minutes** |
| Concurrency | 1 listener, 1 thread | 4 concurrent chunks, unbounded across pods |
| OOM kills since rollout | routine | **zero** |
| Failure blast radius | the entire job | one chunk, retried automatically |
| Transaction profile | one multi-minute mega-transaction | hundreds of sub-second commits |

Same file, a fifth of the time, a tenth of the memory — and the gigabyte that never used to finish now clears in 13 minutes.

The concurrency number is the one I'd draw attention to, because it's the least interesting-looking and the most important: 4 is a configured value, not a ceiling. Chunks are independent and self-describing, so any pod can process any chunk of any file. Throughput is now a function of how many pods we run — which is to say it's an ops decision, not an architecture problem.

---

## Takeaways

1. **Make the chunk the unit of everything** — storage, messaging, processing, retries. Bound the unit of work and you've bounded memory, timeouts, and failure recovery too.
2. **Never let an ORM near a large LOB.** It will faithfully hand you the entire thing, every time. Query IDs, fetch payloads one at a time, let each die young.
3. **Keep chunks self-similar.** Pull headers and metadata out of the data stream so identical, stateless code can process any chunk.
4. **Durable row-level progress buys you idempotency cheaply.** Writing the work down before doing it turns an at-least-once queue into effectively-once processing — and hands you completion detection and output ordering as side effects.
5. **Hide storage you're not proud of behind an interface.** Our chunks live in a database because production said so. Because nothing else knows that, changing it is a swap — not a rewrite.

Most "we need more memory" problems are really "we're holding things we don't need to hold" problems. The file never needed to be in memory. It just needed somewhere to flow through.

*Built with a team that pressure-tested the design hard enough to make it production-worthy — the one-sentence idea was the easy part.*

---

*Thanks for reading. If you've solved this differently — object stores, temp files, reactive streams — I'd like to hear how it went. Find me on [LinkedIn](https://www.linkedin.com/in/kumar-manas-7b755b38/).*
