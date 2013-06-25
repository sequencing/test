Hardware requirements   {#hardware}
---------------------
As a ballpark figure, if there is Y GBs of BCL data, then iSAAC roughly does the following:

* Reads 2xY GBs of BCL files
* Reads 50 GB of sorted reference (for human)
* Writes 4xY GBs of Temporary data  
* Reads 4xY GBs of Temporary Data
* Writes Y GBs of BAM Data

As a rule of thumb, given a reasonably high end modern CPU and enough memory for a lane of BCL files plus the reference, then the scratch storage should be able to do over 200 MB/s to avoid IO dominating the processing time and preferable over 500 MB/s.
