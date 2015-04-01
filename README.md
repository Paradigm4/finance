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
The data consist of many columns, of which we will only use the first few for
example purposes. The best way to load delimited text data into SciDB uses
the https://github.com/Paradigm4/load_tools plugin--see the plugin project
page for installation details.
