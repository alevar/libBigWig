A C library for reading/parsing local and remote BigWig files. While Kent's source code is free to use for these purposes, it's really inappropriate as library code since it has the unfortunate habit of calling `exit()` whenever there's an error. If that's then used inside of something like python then the python interpreter gets killed. This library is aimed at resolving these sorts of issues and should also use more standard things like curl and has a friendlier license to boot.

Documentation is automatically generated by doxygen and can be found under `docs/html` or online [here](https://cdn.rawgit.com/dpryan79/libBigWig/master/docs/html/index.html).

#Example

The only functions and structures that end users need to care about are in "bigWig.h". Below is a commented example. You can see the files under `test/` for further examples.

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

##Writing example
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

#A note on statistics

The results of `min`, `max`, and `mean` should be the same as those from `BigWigSummary`. `std` and `coverage`, however, may differ due to Kent's tools producing incorrect results (at least for `coverage`, though the same appears to be the case for `std`).

#To do
 - [ ] Profile code, since this is likely slow in places.
 - [ ] Restructure the headers so there's only one that needs to be included/installed.
