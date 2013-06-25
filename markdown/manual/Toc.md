[TOC]

# Hardware requirements
As a ballpark figure, if there is Y GBs of BCL data, then iSAAC roughly does the following:

* Reads 2xY GBs of BCL files
* Reads 50 GB of sorted reference (for human)
* Writes 4xY GBs of Temporary data  
* Reads 4xY GBs of Temporary Data
* Writes Y GBs of BAM Data

As a rule of thumb, given a reasonably high end modern CPU and enough memory for a lane of BCL files plus the reference, 
then the scratch storage should be able to do over 200 MB/s to avoid IO dominating the processing time and preferable over 500 MB/s.


# Sorted Reference
iSAAC requires a pre-processed reference to do the alignment. The pre-processing extracts all possible 32-mers from the 
reference genome and stores them in the format that is easily accessible to [isaac-align](#isaac-align) 
along with the metadata. The metadata file also keeps the absolute path to the original .fa file and isaac-align uses this file. 
It is important to ensure that this file is available in its original location at the time isaac-align is being run.

As the metadata uses absolute paths to reference files, manually copying or moving the sorted refernce is not recommended. 
Instead, using the [isaac-pack-reference](#isaac-pack-reference)/[isaac-unpack-reference](#isaac-unpack-reference) tool pair is advised.

In order to prepare a reference from an .fa file, use [isaac-sort-reference](#isaac-sort-reference).


# Toolkit Reference


## isaac-align
### Usage
```
isaac-align -r <reference> -b <base calls> -m <memory limit> [optional arguments]
```


## isaac-sort-reference
> **NOTE:** Available RAM could be a concern when sorting big genomes. Human genome reference sorting will require ~150 gigabytes of RAM.

### Usage
```
isaac-sort-reference [options]
```

Time highly depends on the size of the genome. For E. coli it takes about 1 minute. For human genomes it takes about 11 
hours on a 24-threaded 2.6GHz system.

### Options
Option       | Description
-------------|-------------------
-h [ --help ]| Print help message
-n [ --dry-run ]|Don't actually run any commands; just print them
-v [ --version ]|Only print version information
-j [ --jobs ] arg (=1)|Maximum number of parallel operations
-w [ --mask-width ] arg (=6) |Number of high order bits to use for splitting the data for parallelization
-g [ --genome-file ] arg|Path to fasta file containing the reference contigs
-o [ --output-directory ] arg (./iSAACIndex.20120704)|Location where the results are stored
