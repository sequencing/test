> **NOTE:** Available RAM could be a concern when sorting big genomes. Human genome reference sorting will require ~150 gigabytes of RAM.

Usage
-----

```
isaac-sort-reference [options]
```

Time highly depends on the size of the genome. For E. coli it takes about 1 minute. For human genomes it takes about 11 hours on a 24-threaded 2.6GHz system.

Options
-------

| Item      |  Value | Qty  |
| :-------- | ------:| :--: |
| Computer  | \$1600 |  5   |
| Phone     |   \$12 |  12  |
| Pipe      |    \$1 | 234  |



Item      | Value
--------- | -----
Computer  | \$1600
Phone     | \$12
Pipe      | \$1

Option       | Description
-------------|-------------------
-h [ --help ]| Print help message

-n [ --dry-run ]
Don't actually run any commands; just print them
-v [ --version ]
Only print version information
-j [ --jobs ] arg (=1)
Maximum number of parallel operations
-w [ --mask-width ] arg (=6) 
Number of high order bits to use for splitting the data for parallelization
-g [ --genome-file ] arg
Path to fasta file containing the reference contigs
-o [ --output-directory ] arg (./iSAACIndex.20120704)
Location where the results are stored
