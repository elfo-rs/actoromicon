# Pattern: ID Generation

It should be noted that ID generation approach which was chosen in your app affects your app's architecture and overall performance.
Here we discuss how to design good ID and show a couple of great ID designs.

## Choosing the domain for your IDs

Despite of wide spread of 64-bit computer architecture several popular technologies still don't support 64-bit unsigned integer numbers completely.
For example several popular versions of Java [have only `i64` type](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html) so you [can't have](https://docs.oracle.com/javase/8/docs/api/java/lang/Long.html#MAX_VALUE) positive integer numbers bigger than \\( 2^{63} - 1 \\) without excess [overhead](https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html).
Therefore a lot of Java-powered apps (e.g. Graylog) have limitations on IDs' domain.

At some external apps you can gain a couple of problems because of IDs less than `1`, e.g. value `0` may be [explicitly forbidden](https://cloud.google.com/datastore/docs/concepts/entities#assigning_your_own_numeric_id).
Avoiding `0` value in IDs simplifies some popular [memory optimizations](https://doc.rust-lang.org/core/num/struct.NonZeroU64.html) and allows you to use zero values instead of `Nullable` columns which can improve performance in some databases (e.g. ClickHouse).

This leads us to the following integer numbers' domain for IDs:

```
MIN = 1
MAX = 2 ^ 63 - 1 =
    = 9_223_372_036_854_775_807 =
    = 0x7fff_ffff_ffff_ffff ≈
    ≈ 9.2e18
```

Probably today you don't need to be compatible with such domain-limiting systems but in most cases such limitations are very easy to withstand and there's no reason for your project not to be ready for integration with such obsolete systems in the future.
Such domain allows you to store \\( 2^{63} - 1 \\) entities which is bigger than \\( 9.2 \dot{} 10^{18} \\) and is enough to uniquely identify entities in almost every possible practical task.
You shouldn't forget though that a good ID generation scheme is just a trade-off between a dense domain usage and robust sharded unique IDs generation algorithm.

That's why we recommend using 63-bit positive integers as IDs.

### Systems with ECMAScript

In theory JSON allows you to use numbers of any length, but in a bunch of scenarios it will be great to process or display your data using ECMAScript (JavaScript, widely spread in web browsers) and it has [only](https://262.ecma-international.org/11.0/#sec-numbers-and-dates) `f64` primitive type for numbers so you [can't have](https://262.ecma-international.org/11.0/#sec-number.max_safe_integer) positive integer numbers bigger than \\( 2^{53} - 1 \\) without excess [overhead](https://262.ecma-international.org/11.0/#sec-ecmascript-language-types-bigint-type).
This leads us to the following integer numbers' domain for IDs in systems with ECMAScript:

```
MIN = 1
MAX = 2 ^ 53 - 1 =
    = 9_007_199_254_740_991 =
    = 0x1f_ffff_ffff_ffff ≈
    ≈ 9.0e12
```

From the other hand you can add an extra serialization step storing your IDs in JSON as strings which will be a decent workaround for this problem.

## Properties of your ID generation algorithm

We call generated IDs sequence _monotonic_ if every subsequent ID is bigger than a previous one as a number.

### Why monotonic IDs are so great

The app should gain parameters for ID generation algorithm after every restart taking into account previously issued IDs.
Monotonic IDs generation allows you to take into account only the most recently generated IDs or don't bother about existing entries at all.

Most of the modern storages allow you to take a notable performance advance if your data was put down in an ordered or partially-ordered form.
The storage lays down the data according to [sorting key](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#mergetree-query-clauses) which [may differ](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#choosing-a-primary-key-that-differs-from-the-sorting-key) from the primary key (ID).
To perform time series queries on partially-ordered data very quickly one can use the cheapest form of [data skipping indices](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#table_engine-mergetree-data_skipping-indexes) — the `minmax` index.

Partially-monotonic data can be compressed using [specialized codecs](https://clickhouse.tech/docs/en/sql-reference/statements/create/table/#create-query-specialized-codecs) very effectively, thanks to storing deltas instead of absolute values.
Effective data compression is able to lower storage consumption and increase DB throughput significantly.

This means that you should have a good reason to use non-monotonic IDs.

### Level of monotonicity

Depending on the length of monotonic segments in the generated IDs sequence we can divide ID generation algorithms to:

- _Fully monotonic_ — generated ID sequence is monotonic during the whole lifetime of the system (see also [IDs produced by DB][ids_produced_by_db]).
- _Periodically monotonic_ — ID generation algorithm reaches the maximal value for ID several times during the lifetime of the system and starts counting from the beginning of the domain.
    This means that the storage would contain a couple of monotonic segments e.g. with data for several months long each (see also [`TraceId`][TraceId]).
- With _sharded monotonicity_ — the app generates IDs monotonically for several segments frequently switching between these segments.
    E.g. every actor generates only monotonic IDs but because of concurrent nature of actors' work the storage should handle a lot of intersecting consecutive IDs inserts (see also [`DecimalId`][DecimalId]).

### Monotonicity source

There're several means to gain monotonic IDs.
Your choice of them would affect application restart strategy:

- [Using DB][ids_produced_by_db].
    Needs atomic [compare-and-set](https://www.scylladb.com/2020/07/15/getting-the-most-out-of-lightweight-transactions-in-scylla/) (ScyllaDB) or auto-increment
- Launch number (e.g. [`DecimalId`][DecimalId]).
- Time (e.g. [`TraceId`][TraceId]).
    Time-series-based (not determined, has advantages limitations).

### The reasons for non-monotonic IDs

If ID generation algorithm shouldn't look predictive because of information security concerns it's a good reason to use [multiplicative hashing](https://en.wikipedia.org/wiki/Hash_function#Multiplicative_hashing) or [random permutations](https://preshing.com/20121224/how-to-generate-a-sequence-of-unique-random-integers/) to generate your IDs.

It's usually a great idea to apply randomness tests to your IDs in such scenarios.

Please note that really secure IDs would require security algorithms (or at least some versions of UUID).
Algorithms mentioned above are for IDs that should just _look random_.

### Common or separate domains for several entities' IDs

Should we use common ID generator for several entities?

In such scenario IDs for various entities will never intersect so you have no chance to successfully query entity A by ID of entity B by mistake and you essentially have “fail fast” approach on entities' querying.

Unfortunately common ID generator for several entities will exhaust ID domain more quickly.

## Great IDs case study

### IDs produced by DB

Probably you did already use `AUTO_INCREMENT` SQL DDL instruction for primary keys.
The general idea with it is to never generate IDs by ourselves and instead rely on the counter inside the database as a single source of truth.
This means that you don't know the IDs of entities you'd like to insert before you actually insert them so you should essentially support separate schema just for entities' “construction stubs” and your app's latency is bound by DB latency by design.
If you need a cycle reference between entities you'd like to insert you're most likely in trouble.
If you need database replication you'll be in trouble [too](https://mariadb.com/kb/en/auto_increment/#replication).

Alternative — is to use atomic compare-and-swap integrated into several DB engines, for example [Cassandra's lightweight transactions](https://stackoverflow.com/a/29391877/697625).
IDs produced by DB are great for storing entities appearing with relatively low frequency (probably less than 10000 per second): like users or accounts.

Such approach gives us [fully monotonic][monotonicity_level] IDs and uses the whole [63-bit space][domain] to store ID's positive value right away.

### `DecimalId`

Probably your application already has some kind of _launch log_ storing information about release ticket in the issue tracking system, launch time, user initiated the launch, etc.
If you have such entity in your system `DecimalId` will probably serve to you really good.
This ID has [sharded monotonicity][monotonicity_level].
The primary idea behind `DecimalId` is to increase ID's legibility by storing its parts in a separate decimal places.

`DecimalId` essentially is:

\\[
\operatorname{decimal\\_id} = \operatorname{counter} \dot{} 10^{10} + \operatorname{generator\\_id} \dot{} 10^{5} + \operatorname{launch\\_id}
\\]

This formula corresponds to the following decimal places distribution table:

| Digits | Description | Range | Source |
| -----: | ----------- | ----- | ------ |
|      9 | `counter` | `1..=922_337_202` | Not synchronized runtime parameter |
|      5 | `generator_id` | `0..=99_999` | Synchronized runtime parameter |
|      5 | `launch_id` | `1..=99_999`, `0` for tests | Externally specified and consistent across the whole system during a single launch |

The code implementing `DecimalId` generation can't use bit shift hack as we did for `TraceId`, but IDs generated using such schema are much more legible.
You're able to read their decimal representation like this: `cc*ggggglllll` (**c**ounter, **g**enerator ID, **l**aunch ID — respectively).
For instance:

```
               ID 14150009200065
                  ccccggggglllll
                   ▲    ▲    ▲
   counter (1415) ─┘    │    │
generator_id (92) ──────┘    │
   launch_id (65) ───────────┘
```

[Maximal possible][domain] ID value is `922_337_203___68_547___75_807` however **`counter`** is limited by `922_337_202` (please note the decrement).
Such trick prevents limiting `generator_id` (next ID component) by `68_547` (now it's limited by `99_999`).
Note that \\( \operatorname{counter}_{min} = 1 \\) to keep the [invariant][domain]: \\( \operatorname{id} \geqslant 1 \\) because every other conmonent of `DecimalId` could be zero.

Every time system produces more than \\( \operatorname{counter}_{max} \\) elements it requests persistent synchronized counter global for the whole system for the next **`generator_id`**.
This increment happens at the every start of the app and only once for every \\( 9.2 \dot{} 10^{8} \\) records so contention on it is negligible.

At every launch of your app **`launch_id`** should be taken from the persistent launch log of the system.
To make deployment more robust we recommend generating `launch_id` outside of the app in the deployment configuration.

`launch_id = 0` should never appear in production records and may only be used for testing.

[DecimalId]: #decimalid
[domain]: #choosing-the-domain-for-your-ids
[dumping]: ./ch04-03-dumping.html
[ids_produced_by_db]: #ids-produced-by-db
[logging]: ./ch04-01-logging.html
[monotonicity_level]: #level-of-monotonicity
[TraceId]: ./ch04-04-tracing.html#traceid
[tracing]: ./ch04-04-tracing.html
