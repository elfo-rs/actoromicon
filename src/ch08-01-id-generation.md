# Pattern: ID Generation

It should be noted that ID generation approach which was chosen in your app affects your app's architecture and overall performance.
Here we discuss how to design good ID and show a couple of great ID designs.

## Designing a great ID

### Choosing the domain for your IDs

Despite of wide spread of 64-bit computer architecture several popular technologies still don't support 64-bit unsigned integer numbers completely.
For example several popular versions of Java [have only `i64` type](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html) so you [can't have](https://docs.oracle.com/javase/8/docs/api/java/lang/Long.html#MAX_VALUE) positive integer numbers bigger than `2 ^ 63 - 1` without excess [overhead](https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html).
Therefore a lot of Java-powered apps (e.g. Graylog) have limitations on IDs' domain.
At some external apps you can gain a couple of problems because of IDs less than `1`, e.g. value `0` may be [explicitly forbidden](https://cloud.google.com/datastore/docs/concepts/entities#assigning_your_own_numeric_id).
This leads us to the following integer numbers' domain for IDs:

|   | Min | Max |
| - | --- | --- |
|   | `1` | `2 ^ 63 - 1` |
| Scientific | `1` | `≈ 9.2e18` |
| Decimal | `1` | `9_223_372_036_854_775_807` |
| Hex | `0x1` | `0x7fff_ffff_ffff_ffff` |

Probably today you don't need to be compatible with such systems but in most cases such limitations are very easy to withstand and there's no reason for your project not to be ready for integration with such obsolete systems in the future.
Such domain allows you to store `2 ^ 63 - 1` entities which is bigger than `9.2e18` and is enough to uniquely identify entities in almost every possible practical task.
You shouldn't forget though that a good ID generation scheme is just a trade-off betweeen a dense domain usage and robust sharded unique IDs generation.
That's why we recommend using 63-bit positive integers as IDs.

#### Systems with ECMAScript

In theory JSON allows you to use numbers of any length, but in a bunch of scenarios it will be great to process or display your data using ECMAScript (JavaScript, widely spread in web browsers) and it [has only `f64` primitive type](https://262.ecma-international.org/11.0/#sec-numbers-and-dates) for numbers so you [can't have](https://262.ecma-international.org/11.0/#sec-number.max_safe_integer) positive integer numbers bigger than `2 ^ 53 - 1` without excess [overhead](https://262.ecma-international.org/11.0/#sec-ecmascript-language-types-bigint-type).
This leads us to the following integer numbers' domain for IDs in systems with ECMAScript:

|   | Min | Max |
| - | --- | --- |
|   | `1` | `2 ^ 53 - 1` |
| Scientific | `1` | `≈ 9.0e12` |
| Decimal | `1` | `9_007_199_254_740_991` |
| Hex | `0x1` | `0x1f_ffff_ffff_ffff` |

From the other hand you can add an extra serialization step storing your IDs in JSON as strings which will be a decent workaround for this problem.

### Properties of your ID

#### Level of monotony

Depending on the length of monotonous segments in the generated IDs sequence we can divide ID generation algorithms to:

- _Fully monotonous_ — generated ID sequence is monotonous during the whole lifetime of the system.
- _Periodically monotonous_ — ID generation algorithm reaches the maximal value for ID several times during the lifetime of the system and starts counting from the beginning of the domain.
    This mean that storage would contain a couple of monotonous segments e.g. several months long each (see also [`TraceId`][TraceId]).
- Applicable to _sharded monotony_ — the app generates IDs monotonously for several segments frequently switching between these segments.
    E.g. every actor generates only monotonous IDs but because of their concurrent work storage should handle a lot of intersecting consecutive IDs inserts (see also [`DecimalId`][DecimalId]).

#### Monotony source

- via DB
- Decimal
- Time

### Why monotonous IDs are so great

Most of the modern storages allow you to take a notable performance advance if your data was put down in an ordered or partially-ordered form.
The storage itself acts as a [clustered index](https://use-the-index-luke.com/sql/glossary/clustered-index) ordered by its primary key.
In most cases choosing of monotonous IDs drastically improves data compression in the DB and allows to perform the most frequent requests to the DB very quickly.
This means that you should have a good reason to use non-monotonous IDs.

#### The reasons for non-monotonous IDs

If ID generation algorithm shouldn't look predictive because of information security concerns it's a good reason to use [random permutations](https://en.wikipedia.org/wiki/Random_permutation) or [multiplicative hashing](https://en.wikipedia.org/wiki/Hash_function#Multiplicative_hashing) to generate your IDs.
It's usually a great idea to apply randomness tests to your IDs in such scenarios.

#### Taking monotonous IDs from the DB

needs atomic compare-and-set (ScyllaDB) or autoincrement

#### Decimal

TODO

#### Time
Timeseries-based (not determined, has advantages limitations). Example: trace ID generation.

## Great IDs case study

### `DecimalId`

### `TraceId`

Uses periodic monotony with a period approx. 1 year

[DecimalId]: #decimalid
[TraceId]: #traceid
