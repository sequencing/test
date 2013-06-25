[TOC]



  Right     Left     Center     Default
-------     ------ ----------   -------
     12     12        12            12
    123     123       123          123
      1     1          1             1


-------------------------------------------------------------
 Centered   Default           Right Left
  Header    Aligned         Aligned Aligned
----------- ------- --------------- -------------------------
   First    row                12.0 Example of a row that
                                    spans multiple lines.

  Second    row                 5.0 Here's another one. Note
                                    the blank line between
                                    rows.
-------------------------------------------------------------


# Hardware requirements

## RAM

When running from Bcl data, for human genome analyses, it is recommended to let iSAAC use 40+ GB of RAM on a 24-threaded 
system. See [tweaks](#tweaks) section for ways to run iSAAC on limited hardware.

## IO

As a ballpark figure, if there is Y GBs of BCL data, then iSAAC roughly does the following:

* Reads 2Y GBs of BCL files
* Reads 50 GB of sorted reference (for human)
* Writes 4Y GBs of Temporary data  
* Reads 4Y GBs of Temporary Data
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

# Tweaks

## Reducing RAM requirement

iSAAC human genome alignment can be run in as little as 32 GB RAM by limiting the amount of compute threads with 
[-j](#isaac-align) parameter.

## Analyzing human genomes on low RAM System

As the human reference requires about 46 gigabytes to stay in RAM, the 48 gigabyte (or smaller) systems are not able to 
keep it in the cache. In this situation it pays to minimize the number of reference scanning passess the MatchFinder need to perform.

The number of tiles processed at a time is usually capped by the --temp-parallel-save parameter, which is set to 16 by 
default. This limit prevents the match files to be excessively fragmented on the systems where temporary data is stored 
on a local hard drive. If the temporary data is stored on a distributed network storage such as Isilon, the fragmentation 
is usually not an issue. In this case --temp-parallel-save 64 will ensure that entire lane (64 is the currently known 
maximum number of tiles per lane produced by an instrument) is loaded into memory for MatchFinder.

On the systems with more than 64 gigabytes of physical RAM, it makes sense to keep the defaults as in this case the 
reference has enough room to stay in the IO cache.

## Reducing Linux swappiness

iSAAC is designed to take the full advantage of the hardware resources available on the processing node. On systems with 
default Linux configuration, this causes the operating system to swap pages out to make room for the IO cache, when the 
iSAAC gets to a memory/IO intenstive stages. Since iSAAC operates on data volumes that normally exceed the amount of RAM 
of an average system, there is little or no benefit from IO-caching the data iSAAC reads and writes. As result, the data 
processing takes longer than it would if the system did not prioritize IO cache over the code and data that is already 
in RAM.

Reducing the value **_/proc/sys/vm/swappiness_** to 10 or lower from default 60 solves the problem.

# Toolkit Reference


## isaac-align

**Usage**

```
isaac-align -r <reference> -b <base calls> -m <memory limit> [optional arguments]
```

Option                       | Description
:----------------------------|:----------------------------------------------------------------------------------------
  -h [ --help ]              |produce help message and exit
  -v [ --version ]           |print program version information
  -b [ --base-calls ] arg    |full path to the base calls directory. Multiple entries allowed.
  --base-calls-format arg    |Multiple entries allowed. Each entry is applied to the corresponding base-calls. Last entry
                             | is applied to all --base-calls that don't have --base-calls-format specified.
                             |  _bam_         : Bam file. All data found in bam file is assumed to come from lane 1 of a 
                             |                  single flowcell.
                             |  _bcl_         : common bcl files, no compression.
                             |  _bcl-gz_      : bcl files are individually compressed and named s_X_YYYY.bcl.gz
                             |  _fastq_       : One fastq per lane/read named lane<X>_read<Y>.fastq and located directly 
                             |                  in the specified base-calls. Use lane<X>_read1.fastq for single-ended data.
                             |  _fastq-gz_    : One compressed fastq per lane/read named lane<X>_read<Y>.fastq.gz and 
                             |                  located directly in the specified base-calls. Use lane<X>_read1.fastq.gz 
                             |                  for single-ended data.


## isaac-sort-reference
> **NOTE:** Available RAM could be a concern when sorting big genomes. Human genome reference sorting will require ~150 gigabytes of RAM.

**Usage**

```
isaac-sort-reference [options]
```

Time highly depends on the size of the genome. For E. coli it takes about 1 minute. For human genomes it takes about 11 
hours on a 24-threaded 2.6GHz system.

**Options**

Option                                               | Description
:----------------------------------------------------|:---------------------------------------------------------------------------
-h [ --help ]                                        | Print help message
-n [ --dry-run ]                                     |Don't actually run any commands; just print them
-v [ --version ]                                     |Only print version information
-j [ --jobs ] arg (=1)                               |Maximum number of parallel operations
-w [ --mask-width ] arg (=6)                         |Number of high order bits to use for splitting the data for parallelization
-g [ --genome-file ] arg                             |Path to fasta file containing the reference contigs
-o [ --output-directory ] arg (./iSAACIndex.\<date\>)|Location where the results are stored

