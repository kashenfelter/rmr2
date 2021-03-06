






* This document responds to several inquiries on data formats and how to get data in and out of the rmr system
* Still more a collection of snippets than anything organized
* Thanks Damien  and @ryangarner for the examples and Koert for conversations on the subject

Internally `rmr` uses R's own serialization in most cases and its own typedbytes extension for some atomic vectors. The goal is to make you forget about representation issues most of the time. But what happens at the boundary of the
system, when you need to get non-rmr data in and out of it? Of course `rmr` has to be able to read and write a variety of formats to be of any use. This is what is available and how to extend it.

## Built in formats

The complete list is:

```
[1] "text"                "json"                "csv"                
[4] "native"              "sequence.typedbytes" "hbase"              
[7] "pig.hive"           
```


1. `text`: for english text. key is `NULL` and value is a string, one per line. Please don't use it for anything else.
1. `json`-ish: it is actually `<JSON\tJSON\n>` so that streaming can tell key and value. This implies you have to escape all newlines and tabs in the JSON part. Your data may not be in this form, but almost any
language has decent JSON libraries. It was the default in `rmr` 1.0, but we'll keep because it is almost standard. Parsed in C for efficiency, should handle large objects.
1. `csv`: A family of concrete formats modeled after R's own `read.table`. See examples below.
1. `native`: based on R's own serialization, it is the default and supports everything that R's `serialize` supports. If you want to know the gory details, it is implemented as an application specific type for the typedbytes format, which is further encapsulated in the sequence file format when writing to HDFS, which ... Dont't worry about it, it just works. Unfortunately, it is written and read by only one package, `rmr` itself.
1. `sequence.typedbytes`: based on specs in HADOOP-1722 it has emerged as the standard for non Java hadoop application talking to the rest of Hadoop. Also implemented in C for efficiency, its underlying data model is different from R's and we tried to map the two systems the best we could.
1. `hbase`: read directly from an Hbase table. Experimental and input only for now. Experimental means that testing has not been as thourough as it should and that the feature could be withdrawn in a minor release.

## The easy way

Specify one of those format strings directly as arguments to `mapreduce`, `from.dfs`, `to.dfs`.

```
mapreduce(input, input.format = "json")
```

## Custom formats light
Use `make.input.format` with a string format argument and additional arguments to specify some variants to that format. Typical example is `csv` which is actually a family of character separated formats with lots of variation in the details. In this case you can call something like
```
mapreduce(input, input.format = make.input.format("csv", sep = "\t"))
```
which says to use a CSV format with a tab separator. For this format the arguments are, with few exceptions, the same as `read.table`. The same is true on the output side with `make.output.format` and the model for the additional arguments is `write.table`.

## Custom formats 
A format is a triple. You can create one with `make.input.format`, for instance:

```r
make.input.format("csv")
```

```
$mode
[1] "text"

$format
function (con, keyval.length) 
{
    df = tryCatch(read.table(file = con, nrows = keyval.length, 
        header = FALSE, ...), error = function(e) {
        if (e$message != "no lines available in input") 
            stop(e$message)
        NULL
    })
    if (is.null(df) || dim(df)[[1]] == 0) 
        NULL
    else keyval(NULL, df)
}
<bytecode: 0x104b20b20>
<environment: 0x104b227f0>

$streaming.format
NULL

$backend.parameters
NULL
```


The `mode` element can be `text` or `binary`. The `format` element is a function that takes a connection, reads `nrows` records and creates a key-value object. The `streaming.format` element is a fully qualified Java class (as a string) that writes to the connection the format function reads from. The default is `TextInputFormat` and also useful is `org.apache.hadoop.streaming.AutoInputFormat`. Once you have these three elements you can pass them to `make.input.format` and get something out that can be used as the `input.format` option to `mapreduce` and the `format`  option to `from.dfs`. On the output side the situation is reversed with the R function acting first and then the Java class doing its thing.


```r
make.output.format("csv")
```

```
$mode
[1] "text"

$format
function (kv, con) 
{
    kv = recycle.keyval(kv)
    k = keys(kv)
    v = values(kv)
    write.table(file = con, x = if (is.null(k)) 
        v
    else cbind(k, v), ..., row.names = FALSE, col.names = FALSE)
}
<bytecode: 0x1049cb118>
<environment: 0x1049cb720>

$streaming.format
NULL

$backend.parameters
NULL
```


R data types natively work without additional effort.


```r
my.data = list(TRUE, list("nested list", 7.2), seq(1:3), letters[1:4], matrix(1:25, nrow = 5,ncol = 5))
```


Put into HDFS:

```r
hdfs.data = to.dfs(my.data)
```

`my.data` needs to be one of: vector, data frame, list or matrix. 
Compute a frequency of object lengths.  Only require input, mapper, and reducer. Note that `my.data` is passed into the mapper, record by
record, as `key = NULL, value = subrange`. 


```r
result = mapreduce(
  input = hdfs.data,
  map = function(k, v) keyval(lapply(v, length), 1),
  reduce = function(k, vv) keyval(k, sum(vv)))

from.dfs(result)
```


However, if using data which was not generated with `rmr` (txt, csv, tsv, JSON, log files, etc) it is necessary to specify an input format. 





To define your own `input.format` (e.g. to handle tsv):



```r
tsv.reader = function(con, nrecs){
  lines = readLines(con, 1)
  if(length(lines) == 0)
    NULL
  else {
    delim = strsplit(lines, split = "\t")
    keyval(
      sapply(delim, 
             function(x) x[1]), 
      sapply(delim, 
             function(x) x[-1]))}} 
## first column is the key, note that column indexes moved by 1
```


Frequency count on input column two of the tsv data, data comes into map already delimited


```r
freq.counts = 
  mapreduce(
    input = tsv.data,
    input.format = tsv.format,
    map = function(k, v) keyval(v[1,], 1),
    reduce = function(k, vv) keyval(k, sum(vv)))
```


Or if you want named columns, this would be specific to your data file


```r
tsv.reader = 
  function(con, nrecs){
    lines = readLines(con, 1)
    if(length(lines) == 0)
      NULL
    else {
      delim = strsplit(lines, split = "\t")
      keyval(
        sapply(delim, function(x) x[1]), 
        data.frame(
          location = sapply(delim, function(x) x[2]),
          name = sapply(delim, function(x) x[3]),
          value = sapply(delim, function(x) x[4])))}}
```


You can then use the list names to directly access your column of interest for manipulations

```r
freq.counts = 
  mapreduce(
    input = tsv.data,
    input.format = tsv.format,
    map = 
      function(k, v) { 
        filter = (v$name == "blarg")
        keyval(k[filter], log(as.numeric(v$value[filter])))},
    reduce = function(k, vv) keyval(k, mean(vv)))                      
```


Another common `input.format` is fixed width formatted data:

```r
fwf.reader <- function(con, nrecs) {  
  lines <- readLines(con, nrecs)  
  if (length(lines) == 0) {
    NULL}
  else {
    split.lines = unlist(strsplit(lines, ""))
    df = 
      as.data.frame(
        matrix(
          sapply(
            split(
              split.lines, 
              ceiling(1:length(split.lines)/field.size)), 
            paste, collapse = ""), 
          ncol = length(fields), byrow = T))
    names(df) = fields
    keyval(NULL, df)}} 
fwf.input.format = make.input.format(mode = "text", format = fwf.reader)
```


Using the text `output.format` as a template, we modify it slightly to write fixed width data without tab seperation:

```r
fwf.writer <- function(kv, con, keyval.size) {
  ser = 
    function(df) 
      paste(
          apply(
            df,
            1, 
            function(x) 
              paste(
                format(
                  x, 
                  width = field.size), 
                collapse = "")), 
          collapse = "\n")
  out = ser(values(kv))
  writeLines(out, con = con)}
fwf.output.format = make.output.format(mode = "text", format = fwf.writer)
```


Writing the `mtcars` dataset to a fixed width file with column widths of 6 bytes and putting into hdfs:

```r
fwf.data <- to.dfs(mtcars, format = fwf.output.format)
```


The key thing to note about `fwf.reader` is the global variable `fields`. In `fields`, we define the start and
end byte locations for each field in the data:

```r
fields <- rmr2:::qw(mpg, cyl, disp, hp, drat, wt, qsec, vs, am, gear, carb) 
field.size = 8
```


Sending 1 line at a time to the map function:

```r
out <- from.dfs(mapreduce(input = fwf.data,
                          input.format = fwf.input.format))
out$val
```


Frequency count on `cyl`:

```r
out <- from.dfs(mapreduce(input = fwf.data,
                          input.format = fwf.input.format,
                          map = function(key, value) keyval(value[,"cyl"], 1),
                          reduce = function(key, value) keyval(key, sum(unlist(value))),
                          combine = TRUE))
df <- data.frame(out$key, out$val)
names(df) <- c("cyl","count")
df
```



To get your data out - say you input file, apply column transformations, add columns, and want to output a new csv file
Just like input.format, one must define an output format function:


```r
csv.writer = function(kv, con){
  cat(
    paste(
      apply(cbind(1:32, mtcars), 
            1, 
            paste, collapse = ","), 
      collapse = "\n"),
    file = con)}
```


And then use that as an argument to `make.output.format`, but why sweat it since the devs have already done the work?


```r
csv.format = make.output.format("csv", sep = ",")
```


This time providing output argument so one can extract from hdfs (cannot hdfs.get from a Rhadoop big data object)


```r
mapreduce(
  input = hdfs.data,
  output = tempfile(),
  output.format = csv.format,
  map = function(k, v){
    # complicated function here
    keyval(1, v)},
  reduce = function(k, vv) {
    #complicated function here
    keyval(k, vv[[1]])})
```


### HBase

Reading from an HBase table (an experimental feature) requires specifying yet another format. We rely on the system to be alredy configured to run java MR jobs on HBase tables, see `help(make.input.format)` for a reference. As you can see in the following, non self-contained snippet, one specifies the name of the table as a the input argument to mapreduce (this is not strictly part of the input format but it seemed a good moment to explain it). Then the `input.format` argument is created with a, as usual `make.input.format` with a first argument equal to `"hbase"`. The second argument, `family.colums`, allows  to select certain columns within certain families of columns. It is represented as a  list of lists, with the labels in the outer lists being the name of the families and the element of the lists being the column names. Then we have two deserialiation-related arguments, one for the keys and one for the cell contents. HBase is agnostic to cell content nature. Lacking a source of metadata or a default encoding, keys and cells are passed to R as raw bytes and the user has to fill in the missing information. We support three deserialization formats: "raw" means that the raw bytes are converted to characters, "native" is the built-in R serialization format and "typedbytes" is a format shared with other Hadoop libraries. It's unlikely that these three options are going to cover all applications, therefore both arguments accept also functions for maximum flexibility. For the keys, the function should take a list of raw vectors, deserialize them and return a list with the deserilizaed object. For the cells, the idea is the same but two additional arguments, which will contain the familiy and column being deserialized, allow for additional flexibility. The last two arguments affect the "shape" of the data frame created with the cell contents and the specific types of the columns. `dense` means that a data frame column will be created for each selected HBase column; missing cells will take the value `NA`. When `dense` is `FALSE` the data frame passed to the map function will have always 4 columns, one for the key, one for the family, one for the column and one for the cell contents. That way we can handle the case of sparse HBase tables that can have even millions of columns. Finally `atomic` means that columns can be `unlist`-ed before being added to the data frame, otherwise they are wrapped in a `I` call and added as they are, to support complex cell types.



```r
from.dfs(
  mapreduce(
    input="blogposts", 
    input.format = 
      make.input.format(
        "hbase", 
        family.columns = 
          list(
            image= list("bodyimage"), 
            post = list("author", "body")), 
        key.deserialize = "raw", 
        cell.deserialize = "raw", 
        dense = T, 
        atomic = T)))
```


This is another example run on a freebase data dump loaded into HBase, we have a family named "name" with a single, unnamed column (I suspect this is not recommended, but it works) and another named freebase with a column named "types", which is itself a comma separated list. One could think of a better deserialization that also parses this into a character vector, one element per type, but it's not what is done here.


```r
freebase.input.format = 
  make.input.format(
    "hbase", 
    family.columns = 
      list(
        name = "", 
        freebase = "types"), 
    key.deserialize = "raw", 
    cell.deserialize = "raw", 
    dense = F, 
    atomic = F)
```


The input format thus defined can be used as any other inputp format, on a table called "freebase"


```r
from.dfs(
  mapreduce(
    input = "freebase",
    input.format = freebase.input.format,
    map = function(k,v) keyval(k[1,], v[1,])))
```

