# Basic financial time series SciDB examples

This note describes basic time series operation in SciDB using simple examples
from finance. You can follow along with your own SciDB deployment by copying
and pasting all the examples.

## Equity trade data examples

This section covers efficiently loading data into SciDB from delimited text
files, filtering data based on coordinate and attribute values, data
aggregation, moving window aggregation, and so-called "as of" inexact temporal
joins that combine data with differing time coordinates using last known
values.

### Load the data

NYSE provides a nice set of freely available sample data from a variety of
sources. We use a single day of US exchange-traded equity trade data obtained
with (issued from a Linux command line on the same machine as SciDB):
```
wget ftp://ftp.nyxdata.com/Historical%20Data%20Samples/TAQ%20NYSE%20Arca%20Trades/EQY_US_ALL_ARCA_TRADE_20130404.csv.gz
```

These data consist of only about 3.8 million trades for 6,480 equity
instruments. We can count the number of lines (including the header line) in
the downloaded file with, for example:
```
zcat EQY_US_ALL_ARCA_TRADE_20130404.csv.gz  | wc -l
3871198
```

We use only use the first few data columns for example purposes. The best way
to load delimited text data into SciDB uses the
https://github.com/Paradigm4/load_tools plugin--see the plugin project page for
installation details. The load process described below also requires the
https://github.com/Paradigm4/superfunpack  plugin for a few time and hash
support functions.

An example SciDB AFL-language load script is available in this project from
https://github.com/Paradigm4/finance/blob/master/load_arca_trade.afl
and can be
run with the downloaded equity trade data as shown in the next example:
```
rm -f /tmp/data.csv
mkfifo /tmp/data.csv
zcat EQY_US_ALL_ARCA_TRADE_20130404.csv.gz > /tmp/data.csv &
iquery -naf load_arca_trade.afl
```
The example creates a data pipe that SciDB loads from, decompresses the data file
into the pipe as a background process (the ampersand), and then runs the
load_arca_trade.afl script.

We summarize key points of the load script next. The lines
```
        parse(
          split( '/tmp/data.csv', 'header=1' ),
          'num_attributes=21', 'attribute_delimiter=,'),
```
use the load_tools "split" operator to quickly split and distribute the input
file among the SciDB instances for parallel parsing. The "parse" operator needs
to know at most how many columns are in the data (21), and how the data are
delimited (by commas in this example).

The output of parse is a virtual 22-attribute SciDB array with attribute names
a0 to a20 corresponding to the indicated 21 data columns, plus an extra
attribute named "error" that indicates if any lines were short or long.
The attributes all have type "string."

The `apply` operator creates new array attributes with more reasonable names,
and converts the data types of some of them. In particular it creates an
attribute names "TIMESTAMP" of type int64 that represents the number of seconds
past midnight of each datum (these data have second resolution). It also uses a
trivial perfect hash from the superfunpack plugin called `dumb_hash` to convert
symbol name to an int64 value. "PRICE" and "VOLUME" attributes are also converted
into float and int64 types.

We need the "TIMESTAMP" and "SYMBOL" values to be of type int64 because we
intend to use them as coordinate axes in the final array, and SciDB coordinates
must by int64. Note that we would typically also have a "DATE" coordinate axis
that indicates year-moth-day, but we omitted that from this example for
simplicity and because these data cover only one day.

Note that we could instead use the SciDB `index_lookup`, `sort` and `uniq`
operators to build an auxiliary dimension array that enumerates "SYMBOL" names.
That approach has two important advantages over the trivial hash used here: The
enumeration index set consists of consecutive integers, and index_lookup can
deal with arbitrary-length symbol names (the trivial hash produces a sparse
integer index set and can only hash names up to seven characters long). We use
the hash approach in these examples to simplify the exposition.

Finally, the script organizes a subset of the loaded and processed attributes
into a 3-dimensional coordinate system indexed by "dummy," "SYMBOL," and
"TIMESTAMP." The "dummy" coordinate axis is a special SciDB *synthetic
dimension* that enumerates data that collide in the other coordinates.  This is
important in these example data because more than one trade may be reported for
a given symbol on a given second. An alternate approach to using a synthetic
dimension is to make the exchange sequence number into a coordinate axis.
The schema of the output trades array is:
```
    <MSG_TYPE: string null,
     SEQUENCE: string null,
     TRADE_DATE: string null,
     ORD_REFERS: string null,
     NASDAQID: string null,
     VOLUME: int64 null,
     PRICE: float null>
    [dummy=0:100000,10000,0,
     SYMBOL=0:*,1,0,
     TIMESTAMP=0:*,100000,0]
```
See the 
https://github.com/Paradigm4/finance/blob/master/load_arca_trade.afl
script for details.

This scipt takes a minute or two to load, redimension and store the data into
a SciDB array named "trades." The final array has the same number of data
points as lines in the original file, less the header line:
```
iquery -aq "op_count(trades)"
{i} count
{0} 3871197
```


###  Filtering data

SciDB can quickly fliter data along the coordinate axes using the `between`
and `cross_join` operators.

Use the `filter` operator to filter on attribute values or on arbitrary
expressions involving attributes and coordinates.

Examples follow.

#### Use between to select all trades between 11am and 1pm, and count the
result.

The "TIMESTAMP" coordinate axis is specified in seconds from midnight, so
we need to translate 11am and 1pm into those coordinates: 11am = 11&#215;3600 = 39600
and 1pm = 13&#215;3600=46800. The query is
```
iquery -aq "op_count(between(trades, null,null,39600,  null,null,46800))"
{i} count
{0} 906214
```
Note that `between` is inclusive.


#### Use cross_join to filter on SYMBOL coordinates

The equity SYMBOL names are encoded as integers by a simple hash in this example
(or more generally enumerated in an auxiliary array). When a SYMBOL coordinate
value is known directly then use between similarly to the above example.

Typically we need to consult a hash or auxiliary array to find coordinates for
given SYMBOL NAMES. In that case, using a SciDB cross_join filter is a good way
to go. Cross_join is a database inner join operation between two arrays along a
subset of their coordinate axes.

The next short example builds a tiny 1-d array with a single entry that contains
the coordinate idex of the symbol "AAPL":
```
iquery -aq "
  redimension(
    build(<SYMBOL:int64>[i=0:0,1,0], dumb_hash('AAPL')),
   <i:int64>[SYMBOL=0:*,100000,0])"

{SYMBOL}     i
{1280328001} 0
```
We can use `cross_join` to join this tiny along the SYMBOL axis with the main
trades array, effectively filtering on all trades with SYMBOL=AAPL (we count
the result):
```
iquery -aq "
op_count(
  cross_join(
    trades as x,
    redimension(
      build(<SYMBOL:int64>[i=0:0,1,0], dumb_hash('AAPL')),
      <i:int64>[SYMBOL=0:*,100000,0]) as y,
    x.SYMBOL, y.SYMBOL))"

{i} count
{0} 20530
```
The example data contain 20,530 AAPL trades. Note that the cross_join filtering
array may contain any number of symbol indices. An important tip to
remember when using cross_join is to keep the smaller array (in data size) on the
right.


#### Usig filter

The SciDB `filter` operator can be used to perform arbitrary filtering
operations on attribute values and on expressions involving coordinate values.
Keep in mind that to achieve this flexibility, filter needs to traverse all the
data, an operation that occurs in parallel but can still be considerably more
expensive than filtering along coordinate values with `between` or
`cross_join`.

The following example counts the number of trades of AAPL that were below
430.00. We first use the above `cross_join` operation to cut the data size
down, then use the `filter` operator.
```
iquery -aq "
op_count(
  filter(
    cross_join(
      trades as x,
      redimension(
        build(<SYMBOL:int64>[i=0:0,1,0], dumb_hash('AAPL')),
        <i:int64>[SYMBOL=0:*,100000,0]) as y,
      x.SYMBOL, y.SYMBOL),
    PRICE < 430.00))"

{i} count
{0} 17899
```
The next example repeats the above query using only filter operations to give
you a sense of the performance advantage of usign cross_join and between. The
timings were performed on a single-computer SciDB configuration with 8
instances.
```
time of the previous query:
real    0m0.154s
user    0m0.019s
sys     0m0.007s

# Now just using filter:
time iquery -aq "
op_count( filter(trades, SYMBOL=1280328001 and PRICE < 430.00))"
{i} count
{0} 17899

real    0m0.803s
user    0m0.018s
sys     0m0.008s
```
Both examples run quickly for this tiny example, but it's still wise to
avoid filter on coordinate indices when possible.


### Basic data aggregation

The `redimension` operator is the best mechanism for computing grouped
statistics in sparse SciDB arrays. Redimension allows us to specify reduction
functions that process colliding data values when the coordinate system of an
array is changed. The output of the reduction functions appear as new array
attributes that contain the aggregated stats.

Here are a few examples:

#### Compute the min and max price for every symbol
We pipe the output through head because there are thousands of symbols, and
also apply a string representation of the SYMBOL index to see the symbol names.
```
iquery -aq "
  apply(
    redimension(trades,
      <low: float null, high: float null> [SYMBOL=0:*,1000,0],
       min(PRICE) as low, max(PRICE) as high),
    name,
    dumb_unhash(SYMBOL))" | head 

{SYMBOL} low,high,name
{65} 40.65,41.46,'A'
{66} 27.85,28.24,'B'
{67} 42.16,42.98,'C'
{68} 58.84,59.56,'D'
{69} 45.02,45.72,'E'
{70} 12.605,12.775,'F'
{71} 18.1,18.275,'G'
{72} 41.49,42.19,'H'
{75} 63.55,64.12,'K'
```


### "Asof" joins

This section illustrates combining two arrays along their time axes and
handling cases when time coordinates don't always line up. The following
process looks up and fills in the last known data values as needed.  This kind
of past-looking fuzzy join is sometimes called an "asof" join.
Stated differently, missing values are imputed by piecewise constant
interpolants as required.

Consider the following tables (1-d SciDB arrays, dimensioned along time):
<table>
<tr><td colspan="2">Table A<td><td colspan="2">Table B
<tr><td>Time<td>Price<td>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<td>Time<td>Event
<tr><td>1<td>50.00<td><td>3<td>split
<tr><td>5<td>25.70<td><td><td>
</table>
