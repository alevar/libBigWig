![Master build status](https://travis-ci.org/dpryan79/libBigWig.svg?branch=master) [![DOI](https://zenodo.org/badge/doi/10.5281/zenodo.45278.svg)](http://dx.doi.org/10.5281/zenodo.45278)

A C library for reading/parsing local and remote bigWig and bigBed files. While Kent's source code is free to use for these purposes, it's really inappropriate as library code since it has the unfortunate habit of calling `exit()` whenever there's an error. If that's then used inside of something like python then the python interpreter gets killed. This library is aimed at resolving these sorts of issues and should also use more standard things like curl and has a friendlier license to boot.

Documentation is automatically generated by doxygen and can be found under [`docs/html`](/docs/html) or online [here](https://cdn.rawgit.com/dpryan79/libBigWig/master/docs/html/index.html).

# Example

The only functions and structures that end users need to care about are in "bigWig.h". Below is a commented example. You can see the files under [`test/`](./test/) for further examples.

```c
#include "bigWig.h"
int main(int argc, char *argv[]) {
    bigWigFile_t *fp = NULL;
    bwOverlappingIntervals_t *intervals = NULL;
    double *stats = NULL;
    if(argc != 2) {
        fprintf(stderr, "Usage: %s {file.bw|URL://path/file.bw}\n", argv[0]);
        return 1;
    }

    //Initialize enough space to hold 128KiB (1<<17) of data at a time
    if(bwInit(1<<17) != 0) {
        fprintf(stderr, "Received an error in bwInit\n");
        return 1;
    }

    //Open the local/remote file
    fp = bwOpen(argv[1], NULL, "r");
    if(!fp) {
        fprintf(stderr, "An error occured while opening %s\n", argv[1]);
        return 1;
    }

    //Get values in a range (0-based, half open) without NAs
    intervals = bwGetValues(fp, "chr1", 10000000, 10000100, 0);
    bwDestroyOverlappingIntervals(intervals); //Free allocated memory

    //Get values in a range (0-based, half open) with NAs
    intervals = bwGetValues(fp, "chr1", 10000000, 10000100, 1);
    bwDestroyOverlappingIntervals(intervals); //Free allocated memory

    //Get the full intervals that overlap
    intervals = bwGetOverlappingIntervals(fp, "chr1", 10000000, 10000100);
    bwDestroyOverlappingIntervals(intervals);

    //Get an example statistic - standard deviation
    //We want ~4 bins in the range
    stats = bwStats(fp, "chr1", 10000000, 10000100, 4, dev);
    if(stats) {
        printf("chr1:10000000-10000100 std. dev.: %f %f %f %f\n", stats[0], stats[1], stats[2], stats[3]);
        free(stats);
    }

    bwClose(fp);
    bwCleanup();
    return 0;
}
```

## Writing example

N.B., creation of bigBed files is not supported (there are no plans to change this).

Below is an example of how to write bigWig files. You can also find this file under [`test/exampleWrite.c`](test/exampleWrite.c). Unlike with Kent's tools, you can create bigWig files entry by entry without needing an intermediate wiggle or bedGraph file. Entries in bigWig files are stored in blocks with each entry in a block referring to the same chromosome and having the same type, of which there are three (see the [wiggle specification](http://genome.ucsc.edu/goldenpath/help/wiggle.html) for more information on this).

```c
#include "bigWig.h"

int main(int argc, char *argv[]) {
    bigWigFile_t *fp = NULL;
    char *chroms[] = {"1", "2"};
    char *chromsUse[] = {"1", "1", "1"};
    uint32_t chrLens[] = {1000000, 1500000};
    uint32_t starts[] = {0, 100, 125,
                         200, 220, 230,
                         500, 600, 625,
                         700, 800, 850};
    uint32_t ends[] = {5, 120, 126,
                       205, 226, 231};
    float values[] = {0.0f, 1.0f, 200.0f,
                      -2.0f, 150.0f, 25.0f,
                      0.0f, 1.0f, 200.0f,
                      -2.0f, 150.0f, 25.0f,
                      -5.0f, -20.0f, 25.0f,
                      -5.0f, -20.0f, 25.0f};
    
    if(bwInit(1<<17) != 0) {
        fprintf(stderr, "Received an error in bwInit\n");
        return 1;
    }

    fp = bwOpen("example_output.bw", NULL, "w");
    if(!fp) {
        fprintf(stderr, "An error occurred while opening example_output.bw for writingn\n");
        return 1;
    }

    //Allow up to 10 zoom levels, though fewer will be used in practice
    if(bwCreateHdr(fp, 10)) goto error;

    //Create the chromosome lists
    fp->cl = bwCreateChromList(chroms, chrLens, 2);
    if(!fp->cl) goto error;

    //Write the header
    if(bwWriteHdr(fp)) goto error;

    //Some example bedGraph-like entries
    if(bwAddIntervals(fp, chromsUse, starts, ends, values, 3)) goto error;
    //We can continue appending similarly formatted entries
    //N.B. you can't append a different chromosome (those always go into different
    if(bwAppendIntervals(fp, starts+3, ends+3, values+3, 3)) goto error;

    //Add a new block of entries with a span. Since bwAdd/AppendIntervals was just used we MUST create a new block
    if(bwAddIntervalSpans(fp, "1", starts+6, 20, values+6, 3)) goto error;
    //We can continue appending similarly formatted entries
    if(bwAppendIntervalSpans(fp, starts+9, values+9, 3)) goto error;

    //Add a new block of fixed-step entries
    if(bwAddIntervalSpanSteps(fp, "1", 900, 20, 30, values+12, 3)) goto error;
    //The start is then 760, since that's where the previous step ended
    if(bwAppendIntervalSpanSteps(fp, values+15, 3)) goto error;

    //Add a new chromosome
    chromsUse[0] = "2";
    chromsUse[1] = "2";
    chromsUse[2] = "2";
    if(bwAddIntervals(fp, chromsUse, starts, ends, values, 3)) goto error;

    //Closing the file causes the zoom levels to be created
    bwClose(fp);
    bwCleanup();

    return 0;

error:
    fprintf(stderr, "Received an error somewhere!\n");
    bwClose(fp);
    bwCleanup();
    return 1;
}
```

# Testing file types

As of version 0.3.0, this library supports accessing bigBed files, which are related to bigWig files. Applications that need to support both bigWig and bigBed input can use the `bwIsBigWig` and `bbIsBigBed` functions to determine if their inputs are bigWig/bigBed files:

```c
...code...
if(bwIsBigWig(input_file_name, NULL)) {
    //do something
} else if(bbIsBigBed(input_file_name, NULL)) {
    //do something else
} else {
    //handle unknown input
}
```

Note that these two functions rely on the "magic number" at the beginning of each file, which differs between bigWig and bigBed files.

# bigBed support

Support for accessing bigBed files was added in version 0.3.0. The function names used for accessing bigBed files are similar to those used for bigWig files.

    Function | Use
    --- | ---
    bbOpen | Opens a bigBed file
    bbGetSQL | Returns the SQL string (if it exists) in a bigBed file
    bbGetOverlappingEntries | Returns all entries overlapping an interval (either with or without their associated strings
    bbDestroyOverlappingEntries | Free memory allocated by the above command

Other functions, such as `bwClose` and `bwInit`, are shared between bigWig and bigBed files. See `test/testBigBed.c` for a full example.

# A note on bigBed entries

Inside bigBed files, entries are stored as chromosome, start, and end coordinates with an (optional) associated string. For example, a "bedRNAElements" file from Encode has name, score, strand, "level", "significance", and "score2" values associated with each entry. These are stored inside the bigBed files as a single tab-separated character vector (char \*), which makes parsing difficult. The names of the various fields inside of bigBed files is stored as an SQL string, for example:

    table RnaElements 
    "BED6 + 3 scores for RNA Elements data "
        (
        string chrom;      "Reference sequence chromosome or scaffold"
        uint   chromStart; "Start position in chromosome"
        uint   chromEnd;   "End position in chromosome"
        string name;       "Name of item"
        uint   score;      "Normalized score from 0-1000"
        char[1] strand;    "+ or - or . for unknown"
        float level;       "Expression level such as RPKM or FPKM. Set to -1 for no data."
        float signif;      "Statistical significance such as IDR. Set to -1 for no data."
        uint score2;       "Additional measurement/count e.g. number of reads. Set to 0 for no data."
        )

Entries will then be of the form (one per line):

    59426	115	-	0.021	0.48	218
    51	209	+	0.071	0.74	130
    52	170	+	0.045	0.61	171
    59433	178	-	0.049	0.34	296
    53	156	+	0.038	0.19	593
    59436	186	-	0.054	0.15	1010
    59437	506	-	1.560	0.00	430611

Note that chromosome and start/end intervals are stored separately, so there's no need to parse them out of string. libBigWig can return these entries, either with or without the above associated strings. Parsing these string is left to the application requiring them and is currently outside the scope of this library.

# Interval/Entry iterators

Sometimes it is desirable to request a large number of intervals from a bigWig file or entries from a bigBed file, but not hold them all in memory at once (e.g., due to saving memory). To support this, libBigWig (since version 0.3.0) supports two kinds of iterators. The general process of using iterators is: (1) iterator creation, (2) traversal, and finally (3) iterator destruction. Only iterator creation differs between bigWig and bigBed files.

Importantly, iterators return results by one or more blocks. This is for convenience, since bigWig intervals and bigBed entries are stored in together in fixed-size groups, called blocks. The number of blocks of entries returned, therefore, is an option that can be specified to balance performance and memory usage.

## Iterator creation

For bigwig files, iterators are created with the `bwOverlappingIntervalsIterator()`. This function takes chromosomal bounds (chromosome name, start, and end position) as well as a number of blocks. The equivalent function for bigBed files is `bbOverlappingEntriesIterator()`, which additionally takes a `withString` argutment, which dictates whether the returned entries include the associated string values or not.

Each of the aforementioned files returns a pointer to a `bwOverlapIterator_t` object. The only important parts of this structure for end users are the following members: `entries`, `intervals`, and `data`. `entries` is a pointer to a `bbOverlappingEntries_t` object, or `NULL` if a bigWig file is being used. Likewise, `intervals` is a pointer to a `bwOverlappingIntervals_t` object, or `NULL` if a bigBed file is being used. `data` is a special pointer, used to signify the end of iteration. Thus, when `data` is a `NULL` pointer, iteration has ended.

## Iterator traversal

Regardless of whether a bigWig or bigBed file is being used, the `bwIteratorNext()` function will free currently used memory and load the appropriate intervals or entries for the next block(s). On error, this will return a NULL pointer (memory is already internally freed in this case).

## Iterator destruction

`bwOverlapIterator_t` objects MUST be destroyed after use. This can be done with the `bwIteratorDestroy()` function.

## Example

A full example is provided in `tests/testIterator.c`, but a small example of iterating over all bigWig intervals in `chr1:0-10000000` in chunks of 5 blocks follows:

```c
iter = bwOverlappingIntervalsIterator(fp, "chr1", 0, 10000000, 5);
while(iter->data) {
    //Do stuff with iter->intervals
    iter = bwIteratorNext(iter);
}
bwIteratorDestroy(iter);
```

# A note on bigWig statistics

The results of `min`, `max`, and `mean` should be the same as those from `BigWigSummary`. `stdev` and `coverage`, however, may differ due to Kent's tools producing incorrect results (at least for `coverage`, though the same appears to be the case for `stdev`). The `sum` method doesn't exist in Kent's tools, so note that if zoom levels are used, that it will multiply the block average by the lesser of the number of bases covered in the block and the number of bases in a block overlapping the desired region.

# Python interface

There are currently two python interfaces that make use of libBigWig: [pyBigWig](https://github.com/dpryan79/pyBigWig) by me and [bw-python](https://github.com/brentp/bw-python) by Brent Pederson. Those interested are encouraged to give both a try!

# Building without remote file access

If you want to compile without remote file access (e.g., you don't have curl installed), then you can append `-DNOCURL` to the `CFLAGS` line in the `Makefile`. You will also need to remove `-lcurl` from the `LIBS` line.
