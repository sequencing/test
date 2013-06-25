[TOC]

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

    isaac-align -r <reference> -b <base calls> -m <memory limit> [optional arguments]

**Options**

    -h [ --help ]                                produce help message and exit
    -v [ --version ]                             print program version information
    -b [ --base-calls ] arg                      full path to the base calls directory. Multiple entries allowed.
    --base-calls-format arg                      Multiple entries allowed. Each entry is applied to the corresponding base-calls. Last entry is applied to all --base-calls that don't have --base-calls-format specified.
                                                   - bam             : Bam file. All data found in bam file is assumed to come from lane 1 of a single flowcell.
                                                   - bcl             : common bcl files, no compression.
                                                   - bcl-gz          : bcl files are individually compressed and named s_X_YYYY.bcl.gz
                                                   - fastq           : One fastq per lane/read named lane<X>_read<Y>.fastq and located directly in the specified base-calls. Use lane<X>_read1.fastq for single-ended data.
                                                   - fastq-gz        : One compressed fastq per lane/read named lane<X>_read<Y>.fastq.gz and located directly in the specified base-calls. Use lane<X>_read1.fastq.gz for single-ended data.
    --default-adapters arg                       Multiple entries allowed. Each entry is associated with the corresponding base-calls. Flowcells that don't have default-adapters provided, don't get adapters clipped in the data. 
                                                 Each entry is a comma-separated list of adapter sequences written in the direction of the reference. Wildcard (* character) is allowed only on one side of the sequence. Entries with * apply only to the alignments on the matching strand. Entries without * apply to all strand alignments and are matched in the order of appearance in the list.
                                                 Examples:
                                                   ACGT*,*TGCA       : Will clip ACGT and all subsequent bases in the forward-strand alignments and mirror the behavior for the reverse-strand alignments.
                                                   ACGT,TGCA         : Will find the following sequences in the reads: ACGT, TGCA, ACGTTGCA  (but not TGCAACGT!) regardless of the alignment strand. Then will attempt to clip off the side of the read that is shorter. If both sides are roughly equal length, will clip off the side that has less matches.
                                                   Standard          : Standard protocol adapters. Same as AGATCGGAAGAGC*,*GCTCTTCCGATCT
                                                   Nextera           : Nextera standard. Same as CTGTCTCTTATACACATCT*,*AGATGTGTATAAGAGACAG
                                                   NexteraMp         : Nextera mate-pair. Same as CTGTCTCTTATACACATCT,AGATGTGTATAAGAGACAG
    -s [ --sample-sheet ] arg                    Multiple entries allowed. Each entry is applied to the corresponding base-calls.
                                                   - none            : process flowcell as if there is no sample sheet
                                                   - default         : use <base-calls>/SampleSheet.csv if it exists. This is the default behavior.
                                                   - <file path>     : use <file path> as sample sheet for the flowcell.
    --barcode-mismatches arg (=1)                Multiple entries allowed. Each entry is applied to the corresponding base-calls. Last entry applies to all the bases-calls-directory that do not have barcode-mismatches specified. Last component mismatch value applies to all subsequent barcode components should there be more than one. Examples: 
                                                   - 1:0             : allow one mismatch for the first barcode component and no mismatches for the subsequent components.
                                                   - 1               : allow one mismatch for every barcode component.
                                                   - 0               : no mismatches allowed in any barcode component. This is the default.
    --realign-gaps arg (=sample)                 For reads overlapping the gaps occurring on other reads, check if applying those gaps reduces mismatch count. Significantly reduces number of false SNPs reported around short indels.
                                                   - no              : no gap realignment
                                                   - sample          : realign against gaps found in the same sample
                                                   - project         : realign against gaps found in all samples of the same project
                                                   - all             : realign against gaps found in all samples
    --bam-gzip-level arg (=1)                    Gzip level to use for BAM
    --bam-header-tag arg                         Additional bam entries that are copied into the header of each produced bam file. Use '' to represent tab separators.
    --expected-bgzf-ratio arg (=1)               compressed = ratio * uncompressed. To avoid memory overallocation during the bam generation, iSAAC has to assume certain compression ratio. If iSAAC estimates less memory than is actually required, it will fail at runtime. You can check how far you are from the dangerous zone by looking at the resident/swap memory numbers for your process during the bam generation. If you see too much showing as 'swap', it is safe to reduce the --expected-bgzf-ratio.
    --bam-exclude-tags arg (=ZX,ZY)              Comma-separated list of regular tags to exclude from the output BAM files. Allowed values are: all,none,AS,BC,NM,OC,RG,SM,ZX,ZY
    --tiles arg                                  Comma-separated list of regular expressions to select only a subset of the tiles available in the flow-cell.
                                                 - to select all the tiles ending with '5' in all lanes: --tiles [0-9][0-9][0-9]5
                                                 - to select tile 2 in lane 1 and all the tiles in the other lanes: --tiles s_1_0002,s_[2-8]
                                                 Multiple entries allowed, each applies to the corresponding base-calls.
    --use-bases-mask arg                         Conversion mask characters:
                                                   - Y or y          : use
                                                   - N or n          : discard
                                                   - I or i          : use for indexing
                                                 
                                                 If not given, the mask will be guessed from the B<config.xml> file in the base-calls directory.
                                                 
                                                 For instance, in a 2x76 indexed paired end run, the mask I<Y76,I6n,y75n> means:
                                                   use all 76 bases from the first end, discard the last base of the indexing read, and use only the first 75 bases of the second end.
    --seeds arg (=auto)                          Seed descriptors for each read, given as a comma-separated list-of-seeds for each read. A list-of-seeds is a colon-separated list of offsets from the beginning of the read. 
                                                 Examples:
                                                   - auto            : automatic choice of seeds based on --semialigned-strategy parameter
                                                   - 0:32,0:32:64    : two seeds on the first read (at offsets 0 and 32) and three seeds on the second read (at offsets 0, 32, and 64) and on subsequent reads.
                                                   - 0:32:64         : three seeds on all the reads (at offsets 0, 32 and 64)
                                                 Note that the last list-of-seeds is repeated to all subsequent reads if there are more reads than there are colon-separated lists-of-seeds.
    --seed-length arg (=32)                      Length of the seed in bases. 32 or 64 are allowed. Longer seeds reduce sensitivity on noisy data but improve repeat resolution. Ultimately 64-mer seeds are recommended for 150bp and longer data
    --first-pass-seeds arg (=1)                  the number of seeds to use in the first pass of the match finder. Note that this option is ignored when the --seeds=auto
    -r [ --reference-genome ] arg                Full path to the reference genome XML descriptor. Multiple entries allowed.Each entry applies to the corresponding --reference-name. The last --reference-genome entry may not have a corresponding --reference-name. In this case the default name 'default' is assumed.
    -n [ --reference-name ] arg                  Unique symbolic name of the reference. Multiple entries allowed. Each entry is associated with the corresponding --reference-genome and will be matched against the 'reference' column in the sample sheet. 
                                                 Special names:
                                                   - unknown         : default reference to use with data that did not match any barcode.
                                                   - default         : reference to use for the data with no matching value in sample sheet 'reference' column.
    -t [ --temp-directory ] arg (="./Temp")      Directory where the temporary files will be stored (matches, unsorted alignments, etc.)
    -o [ --output-directory ] arg (="./Aligned") Directory where the final alignment data be stored
    -j [ --jobs ] arg (=24)                      Maximum number of compute threads to run in parallel
    --input-parallel-load arg (=64)              Maximum number of parallel file read operations for --base-calls
    --temp-parallel-load arg (=8)                Maximum number of parallel file read operations for --temp-directory
    --temp-parallel-save arg (=64)               Maximum number of parallel file write operations for --temp-directory
    --output-parallel-save arg (=8)              Maximum number of parallel file write operations for --output-directory
    --repeat-threshold arg (=10)                 Threshold used to decide if matches must be discarded as too abundant (when the number of repeats is greater or equal to the threshold)
    --shadow-scan-range arg (=-1)                -1     - scan for possible mate alignments between template min and max
                                                 >=0    - scan for possible mate alignments in range of template median += shadow-scan-range
    --neighborhood-size-threshold arg (=0)       Threshold used to decide if the number of reference 32-mers sharing the same prefix (16 bases) is small enough to justify the neighborhood search. Use large enough value e.g. 10000 to enable alignment to positions where seeds don't match exactly.
    --verbosity arg (=2)                         Verbosity: FATAL(0), ERRORS(1), WARNINGS(2), INFO(3), DEBUG(4) (not supported yet)
    --start-from arg (=Start)                    Start processing at the specified stage:
                                                   - Start            : don't resume, start from beginning
                                                   - MatchFinder      : same as Start
                                                   - MatchSelector    : skip match identification, continue with template selection
                                                   - AlignmentReports : regenerate alignment reports and bam
                                                   - Bam              : resume at bam generation
                                                   - Finish           : Same as Bam.
                                                   - Last             : resume from the last successful step
                                                 Note that although iSAAC attempts to perform some basic validation, the only safe option is 'Start' The primary purpose of the feature is to reduce the time required to diagnose the issues rather than be used on a regular basis.
    --stop-at arg (=Finish)                      Stop processing after the specified stage is complete:
                                                   - Start            : perform the first stage only
                                                   - MatchFinder      : same as Start
                                                   - MatchSelector    : don't generate alignment reports and bam
                                                   - AlignmentReports : don't perform bam generation
                                                   - Bam              : finish when bam is done
                                                   - Finish           : stop at the end.
                                                   - Last             : perform up to the last successful step only
                                                 Note that although iSAAC attempts to perform some basic validation, the only safe option is 'Finish' The primary purpose of the feature is to reduce the time required to diagnose the issues rather than be used on a regular basis.
    --ignore-neighbors arg (=0)                  When not set, MatchFinder will ignore perfect seed matches during single-seed pass, if the reference k-mer is known to have neighbors.
    --ignore-repeats arg (=0)                    Normally exact repeat matches prevent inexact seed matching. If this flag is set, inexact matches will be considered even for the seeds that match to repeats.
    --mapq-threshold arg (=0)                    Threshold used to filter the templates based on their mapping quality: the BAM file will only contain the templates with a mapping quality greater than or equal to the threshold. Templates (or fragments) with a mapping quality of 4 or more are guaranteed to be uniquely aligned. Those with a mapping quality of 3 or less are either mapping to repeat regions or have a large number of errors.
    --pf-only arg (=1)                           When set, only the fragments passing filter (PF) are generated in the BAM file
    --allow-empty-flowcells arg (=0)             Avoid failure when some of the --base-calls contain no data
    --scatter-repeats arg (=0)                   When set, extra care will be taken to scatter pairs aligning to repeats across the repeat locations 
    --base-quality-cutoff arg (=25)              3' end quality trimming cutoff. Value above 0 causes low quality bases to be soft-clipped. 0 turns the trimming off.
    --variable-read-length arg (=0)              Unless set, iSAAC will fail if the length of the sequence changes between the records of a fastq or a bam file.
    --cleanup-intermediary arg (=0)              When set, iSAAC will erase intermediate input files for the stages that have been completed. Notice that this will prevent resumption from the stages that have their input files removed. --start-from Last will still work.
    --ignore-missing-bcls arg (=0)               When set, missing bcl files are treated as all clusters having N bases for the corresponding tile cycle. Otherwise, encountering a missing bcl file causes the analysis to fail.
    --ignore-missing-filters arg (=0)            When set, missing filter files are treated as if all clusters pass filter for the corresponding tile. Otherwise, encountering a missing filter file causes the analysis to fail.
    --keep-unaligned arg (=back)                 Available options:
                                                  - discard          : discard clusters where both reads are not aligned
                                                  - front            : keep unaligned clusters in the front of the BAM file
                                                  - back             : keep unaligned clusters in the back of the BAM file
    --pre-sort-bins arg (=1)                     Unset this value if you are working with references that have many contigs (1000+)
    --semialigned-gap-limit arg (=100)           The maximum length of the gap that can be introduced to minimize mismatches in a semialigned read. This is a separate algorithm from Smith-Waterman gapped alignment. use --semialigned-gap-limit 0 to disable this functionality.
    --clip-semialigned arg (=1)                  When set, reads have their bases soft-clipped on either sides until a stretch of 5 matches is found
    --clip-overlapping arg (=1)                  When set, the pairs that have read ends overlapping each other will have the lower-quality end soft-clipped.
    --gapped-mismatches arg (=5)                 Maximum number of mismatches allowed to accept a gapped alignment.
    --avoid-smith-waterman arg (=0)              When set, heuristics applied to avoid executing costly smith-waterman on sequences that are unlikely to produce gaps
    --gap-scoring arg (=bwa)                     Gapped alignment algorithm parameters:
                                                  - eland            : equivalent of 2:-1:-15:-3:-25
                                                  - bwa              : equivalent of 0:-3:-11:-4:-20
                                                  - m:mm:go:ge:me:gl : colon-delimited string of values where:
                                                      m              : match score
                                                      mm             : mismatch score
                                                      go             : gap open score
                                                      ge             : gap extend score
                                                      me             : min extend score (all gaps reaching this score will be treated as equal)
    --dodgy-alignment-score arg (=0)             Controls the behavior for templates where alignment score is impossible to assign:
                                                  - Unaligned        : marks template fragments as unaligned
                                                  - 0-254            : exact MAPQ value to be set in bam
                                                  - Unknown          : assigns value 255 for bam MAPQ. Ensures SM and AS are not specified in the bam
    --realign-vigorously arg (=0)                If set, the realignment result will be used to search for more gaps and attempt another realignment, effectively extending the realignment over multiple deletions not covered by the original alignment.
    --realign-dodgy arg (=0)                     If not set, the reads without alignment score are not realigned against gaps found in other reads.
    --realigned-gaps-per-fragment arg (=1)       An estimate of how many gaps the realignment will introduce into each fragment.
    --keep-duplicates arg (=1)                   Keep duplicate pairs in the bam file (with 0x400 flag set in all but the best one)
    --mark-duplicates arg (=1)                   If not set and --keep-duplicates is set, the duplicates are not discarded and not flagged.
    --single-library-samples arg (=1)            If set, the duplicate detection will occur across all read pairs in the sample. If not set, different lanes are assumed to originate from different libraries and duplicate detection is not performed across lanes.
    --bin-regex arg (=all)                       Define which bins appear in the output bam files
                                                 all                   : Include all bins in the bam and all contig entries in the bam header.
                                                 skip-empty             : Include only the contigs that have aligned data.
                                                 REGEX                 : Is treated as comma-separated list of regular expressions. Bam files will be filtered to contain only the bins that match by the name.
    --memory-control arg (=off)                  Define the behavior in case unexpected memory allocations are detected: 
                                                   - warning         : Log WARNING about the allocation.
                                                   - off             : Don't monitor dynamic memory usage.
                                                   - strict          : Fail memory allocation. Intended for development use.
    -m [ --memory-limit ] arg (=0)               Limits major memory consumption operations to a set number of gigabytes. 0 means no limit, however 0 is not allowed as in such case iSAAC will most likely consume all the memory on the system and cause it to crash. Default value is taken from ulimit -v.
    -c [ --cluster ] arg                         Restrict the alignment to the specified cluster Id (multiple entries allowed)
    --tls arg                                    Template-length statistics in the format 'min:median:max:lowStdDev:highStdDev:M0:M1', where M0 and M1 are the numeric value of the models (0=FFp, 1=FRp, 2=RFp, 3=RRp, 4=FFm, 5=FRm, 6=RFm, 7=RRm)
    --stats-image-format arg (=gif)              Format to use for images during stats generation
                                                  - gif        : produce .gif type plots
                                                  - none       : no stat generation
    --buffer-bins arg (=1)                       If set, MatchSelector will buffer bin data before writing it out. If not set, MatchSelector will keep an open file handle per bin and write data into corresponding bins as it appears. This option requires extra RAM, but improves performance on some file systems.
    --qscore-bin arg (=0)                        Toggle QScore binning, this will be applied to the data after it is loaded and before processing
    --qscore-bin-values arg                      Overwrite the default QScore binning values.  Default bins are 0:0,1:1,2-9:6,10-19:15,20-24:22,25-29:27,30-34:33,35-39:37,40-63:40.  Identity bins 1:1,2:2,3:3,4:4,5:5,6:6,7:7,8:8,9:9,10:10,11:11,12:12,13:13,14:14,15:15,16:16,17:17,18:18,19:19,20:20,21:21,22:22,23:23,24:24,25:25,26:26,27:27,28:28,29:29,30:30,31:31,32:32,33:33,34:34,35:35,36:36,37:37,38:38,39:39,40:40,41:41



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

