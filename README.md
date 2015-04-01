# Basic financial time series SciDB examples

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
https://github.com/Paradigm4/finance/load_arca_trade.afl  and can be
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


