# Pattern: ID Generation

It should be noted that ID generation approach which was chosen in your app affects your app's architecture and overall performance.
Here we discuss how to design good ID and show a couple of great ID designs.

## Choosing the domain for your IDs

Despite of wide spread of 64-bit computer architecture several popular technologies still don't support 64-bit unsigned integer numbers completely.
For example several popular versions of Java [have only `i64` type](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html) so you [can't have](https://docs.oracle.com/javase/8/docs/api/java/lang/Long.html#MAX_VALUE) positive integer numbers bigger than \\( 2\^{63} - 1 \\) without excess [overhead](https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html).
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
Such domain allows you to store \\( 2\^{63} - 1 \\) entities which is bigger than \\( 9.2 \dot{} 10\^{18} \\) and is enough to uniquely identify entities in almost every possible practical task.
You shouldn't forget though that a good ID generation scheme is just a trade-off between a dense domain usage and robust sharded unique IDs generation algorithm.

That's why we recommend using 63-bit positive integers as IDs.

### Systems with ECMAScript

In theory JSON allows you to use numbers of any length, but in a bunch of scenarios it will be great to process or display your data using ECMAScript (JavaScript, widely spread in web browsers) and it [has only `f64` primitive type](https://262.ecma-international.org/11.0/#sec-numbers-and-dates) for numbers so you [can't have](https://262.ecma-international.org/11.0/#sec-number.max_safe_integer) positive integer numbers bigger than \\( 2\^{53} - 1 \\) without excess [overhead](https://262.ecma-international.org/11.0/#sec-ecmascript-language-types-bigint-type).
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

We call generated IDs sequence _monotonic_ if every subsequent ID is bigger than previous one as a number.

### Why monotonic IDs are so great

The app should gain parameters for ID generation algorithm after every restart taking into account previously issued IDs.
Monotonic IDs generation allows you to take into account only the most recently generated IDs or don't bother about existing entries at all.

Most of the modern storages allow you to take a notable performance advance if your data was put down in an ordered or partially-ordered form.
The storage itself acts as a [clustered index](https://use-the-index-luke.com/sql/glossary/clustered-index) ordered by its primary key.
To perform requests to partially-ordered data very quickly one can use the cheapest form of [data skipping indices](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/#available-types-of-indices) — the `minmax` index.

Partially-monotonic data can be compressed using [specialized codecs](https://clickhouse.tech/docs/en/sql-reference/statements/create/table/#create-query-specialized-codecs) very effectively, thanks to storing deltas instead of absolute values.
Effective data compression is able to lower storage consumption and increase DB throughput significantly.

This means that you should have a good reason to use non-monotonic IDs.

### Level of monotonicity

Depending on the length of monotonic segments in the generated IDs sequence we can divide ID generation algorithms to:

- _Fully monotonic_ — generated ID sequence is monotonic during the whole lifetime of the system (e.g. [`ConfiguredId`][ConfiguredId]).
- _Periodically monotonic_ — ID generation algorithm reaches the maximal value for ID several times during the lifetime of the system and starts counting from the beginning of the domain.
    This means that the storage would contain a couple of monotonic segments e.g. with data for several months each (e.g. [`TraceId`][TraceId]).
- With _sharded monotonicity_ — the app generates IDs monotonically for several segments frequently switching between these segments.
    E.g. every actor generates only monotonic IDs but because of actors' concurrent work the storage should handle a lot of intersecting consecutive IDs inserts (e.g. [`DecimalId`][DecimalId]).

### Monotonycity source

There're several means to gain monotonic IDs.
Your choice of them would affect application restart strategy:

- via DB (e.g. [`ConfiguredId`][ConfiguredId]).
    Needs atomic [compare-and-set](https://www.scylladb.com/2020/07/15/getting-the-most-out-of-lightweight-transactions-in-scylla/) (ScyllaDB) or auto-increment
- Launch number (e.g. [`DecimalId`][DecimalId]).
- Time (e.g. [`TraceId`][TraceId]).
    Time-series-based (not determined, has advantages limitations).

### The reasons for non-monotonic IDs

If ID generation algorithm shouldn't look predictive because of information security concerns it's a good reason to use [random permutations](https://en.wikipedia.org/wiki/Random_permutation) or [multiplicative hashing](https://en.wikipedia.org/wiki/Hash_function#Multiplicative_hashing) to generate your IDs.
It's usually a great idea to apply randomness tests to your IDs in such scenarios.

Please note that really secure IDs would require security algorithms (or at least some versions of UUID).
Algorithms mentioned above are for IDs that should just _look random._

## Great IDs case study

### `ConfiguredId`

This ID is [fully monotonic][monotonicity_level].
Probably you've always used `AUTOINCREMENT` SQL instruction for primary keys and relied on the database ID generation.
Its primary idea is to never generate IDs by ourselves and instead 
Such IDs use the whole [63-bit space][domain] to store ID value right away.

### `DecimalId`

This ID has [sharded monotonicity][monotonicity_level].

| Digits | Description | Range |
| ------ | ----------- | ----- |
| 1 | [Reserved][domain] | `0` |
| 9 | `counter` | `1..=922_337_202` |
| 5 | `generator_id` | `0..=99_999` |
| 2 | `node_no` | `0..=99` |
| 3 | `launch_no` | `1..=999`, `0` for tests  |

### `TraceId`

This ID is [periodically monotonic][monotonicity_level].
Uses periodic monotonicity with a period approx. 1 year

From MSB to LSB:

| Bits | Description | Range |
| ---- | ----------- | ----- |
| 1 | [Reserved][domain] | `0` |
| 25 | `timestamp` in secs | `0..=33_554_431` |
| 5 | `component` | `0..=31` |
| 15 | `node_no` | `0..=32_767` |
| 18 | `counter` | `1..=262_143` |

[ConfiguredId]: #configuredid
[DecimalId]: #decimalid
[domain]: #choosing-the-domain-for-your-ids
[monotonicity_level]: #level-of-monotonicity
[TraceId]: #traceid
