# Anatomy of the CouchDB changes feed

### The mystery

Let's imagine the following changes to the database. There are two documents, **A** and **B**:

| _id | seq | rev | winner? |
| ---- | --- | ---- | --- |
| A | 1  | 1-a |  |
| B | 2  | 1-b| &#10003;  |
| A | 3  | 2-aa |  	&#10003; |

If you ask for `changes()`, you will see:

```json
{"results":[
{"seq":2,"id":"b","changes":[{"rev":"1-b"}]},
{"seq":3,"id":"a","changes":[{"rev":"2-aa"}]}
],
"last_seq":3}
```

So the first **A** revision is basically forgotten, and only the second one is given. Also notice that it's given *after* the **B** revision, and note that the `last_seq` intuitively refers to the final given `seq`.

But what if the 2nd **A** revision is a non-winner? I.e. what if non-winning revisions are pushed after winning revisions?

Let's make changes like this:

| _id | seq | rev | deleted? | winner? |
| ---- | --- | ---- | --- | --- |
| A | 1  | 1-a | |   |
| A | 2  | 2-aa | |   	&#10003; |
| B | 3  | 1-b| |  &#10003;  |
| A | 4  | 2-aaa | &#10003; |  |

Now if we ask for `changes()`, we get:

```json
{"results":[
{"seq":3,"id":"b","changes":[{"rev":"1-b"}]},
{"seq":4,"id":"a","changes":[{"rev":"2-aa"}]}
],
"last_seq":4}
```

The plot thickens. Even though seq `4` actually refers to a deleted leaf, it's given for both the final `seq` and the `last_seq`!

What about if we `limit` to only the first revision? This should only give us seq `1`, right? Let's try `changes({limit: 1})`

```json
{"results":[
{"seq":3,"id":"b","changes":[{"rev":"1-b"}]}
],
"last_seq":3}
```

WTF? All of the **A** revisions are totally skipped over, and we get **B** instead!

### The explanation

OK, so here is what's going on.

When you tell CouchDB to fetch changes, it actually iterates seq-by-seq through the database, but for each row it encounters, it skips *any seqs less than the latest seq*. Notice that I said "latest seq," not "winning seq." The winning seq matters for things like `{include_docs: true}` (where the winner is fetched), but it doesn't matter for the purpose of ordering.

The other weird thing about this is how it affects `since` and `last_seq`. If you provide e.g. `{since: 3}`, you will get:

```json
{"results":[
{"seq":4,"id":"a","changes":[{"rev":"2-aa"}]}
],
"last_seq":4}
```

So basically CouchDB starts reading *after* the 3 seq (exclusive, not inclusive) and provides any changes it finds from there on out. But importantly, the "seq" in this case refers to the *latest* seq, not the original or winning seq. E.g. if you do `{since: 1, limit: 1}`, you might expect to get **A** (because of the winning 2 seq), but in fact you get **B**:

```json
{"results":[
{"seq":3,"id":"b","changes":[{"rev":"1-b"}]}
],
"last_seq":3}
```

### last_seq

This also gets weird with `last_seq`. Let's try `since=4` just to see what it does:

```json
{"results":[

],
"last_seq":4}
```

WTF? OK, how about `since=1337`:

```json
{"results":[

],
"last_seq":1337}
```

OK Couch, now you're just messing with me.

Actually the answer is very simple; `last_seq` just refers to the maximum of either

* 0
* the `since` (if provided)
* the `seq` of the final row in the results (if results are present)

This is designed with replication in mind. At the end of each `changes()` batch in replication, you want to be able to tell the client, "OK, I've given you all changes since `since`, and the next time you talk to me, you can tell me to ask for changes since `last_seq`." However, if a pathological client goes out of whack and starts asking for `since=1337`, the source DB just dutifully echoes the `since`, which can be bad.

### Other stuff

There's also `style=all_docs`, which basically just says that all leaf revisions need to be returned in the `changes` list. The rule for this is that it has to give the same list regardless of where you are in the sequence list, i.e. it doesn't matter if the current seq is a winner/non-winner.

There's also `descending=true`, which is really weird because 1) `since` is ignored, and 2) `last_seq` doesn't make any sense to me (yet). Also it doesn't really seem to matter, since it's not used in the replicator and I'm not sure what usefulness it would have in userland.

There's also the fact that `seq`s are not necessarily monotonically incrementing integers in BigCouch/CouchDB 2.0/Cloudant, but that's OK; internally to PouchDB we can just assume they're integers, because we do it CouchDB 1.0 style.

Although we don't do it *exactly* CouchDB 1.0 style, because CouchDB 1.0 has this weird pattern where any revisions at the same level of the revision tree actually end up getting the same `seq`s. I assume this has to do with CouchDB's internal structure; in our case we just use IndexedDB's `auto_increment` (or the equivalent in WebSQL/LevelDB), and it's fine because CouchDB doesn't require that revisions at the same tree level all have the same `seq`.