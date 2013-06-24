iSAAC requires a pre-processed reference to do the alignment. The pre-processing extracts all possible 32-mers from the reference genome and stores them in the format that is easily accessible to [isaac-align](toolkitReference/isaac-align.md) along with the metadata. The metadata file also keeps the absolute path to the original .fa file and isaac-align uses this file. It is important to ensure that this file is available in its original location at the time isaac-align is being run.

As the metadata uses absolute paths to reference files, manually copying or moving the sorted refernce is not recommended. Instead, using the [isaac-pack-reference](tookitReference/isaac-pack-reference.md)/[isaac-unpack-reference](tookitReference/isaac-unpack-reference.md) tool pair is advised.

In order to prepare a reference from an .fa file, use [isaac-sort-reference](toolkitReference/isaac-sort-reference.md).
