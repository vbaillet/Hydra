##Comment
This is a mix between several sources of info, incl. https://code.google.com/archive/p/hydra-sv/

Hydra-Multi - an SV discovery tool that incorporates hundreds of samples
=======================================================================

#Overview


1. Generate a config file.
==========================
=======
Hydra-Multi is a paired-end read structural variant discovery tool that is capable of integrating signals from hundreds of samples. Note that Hydra-Multi is also compatible with MP data, provided an additional step meant to correct insert size data calculation.

#Installation
Below are the requirements and instructions for installation of Hydra-Multi. 

##Requirements
1. [samtools](http://samtools.sourceforge.net/)
2. [pysam](https://code.google.com/p/pysam/)
3. `set ulimit -f 16384`

The ulimit determines the number of open file handles on a system.  
This number must be larger than 4*number of possible chromosome-chromosome combinations in the respective reference.  
For the human reference (hg19 b37), 16384 is the recommended ulimit. No need to do so for Arabidopsis genome (60 combinations total if ChrC/M are set aside).

###Installing
	git clone https://github.com/arq5x/Hydra
	cd Hydra
	make 
	chmod +x scripts/*
	sudo cp scripts/* /usr/local/bin
	sudo cp bin/* /usr/local/bin

###Testing Hydra-Multi
	chmod +x hydra-multi.sh
	./hydra-multi.sh test
	
#Running Hydra-Multi
A wrapper script (hydra-multi.sh) can be used to automatiically run Hydra-Multi or each step may be performed manually. Both the automatic and manual executions begin by creating a stub file. 

###0. Generate a stub file.
==========================
Start with a simple config file "stub" such as the one below. The /path/to/bamfile will be the directory in which all the output files will written. It is preferable to dedicate one directory per sample. 

    $ cat config.stub.txt
    sample1	/full/path/to/file/sample1.pos.bam
    sample2	/full/path/to/file/sample2.pos.bam
    sample3	/full/path/to/file/sample3.pos.bam

##Automatic Execution
hydra-multi.sh can then then be used to execute subsequent steps:

	./hydra-multi.sh run config.stub.txt

To obtain a parameter list for using the run script:

	$./hydra-multi.sh run -h
	
	usage:   hydra-multi.sh run [options] <stub_file>
	
	positional arguments:
		stub file
			the stub file to create the configuration file, example on https://github.com/arq5x/Hydra
	options:
		-t INT	Number of threads to use. [Default: 2]
		-p INT	The punt parameter for assembly, the maximum read depth allowed. [Default: 10]
		-o STR	The stub for the output file names

	
##Manual Execution 

###1. Generate a config file.
==========================

HydraMulti needs a configuration file documenting the sample/libraries and the
paths to their respective BAM files that will be input to SV discovery process.

The `make_hydra_config.py` script will inspect the alignments in each sample's
BAM file to automatically create a complete config file documenting the
statistics of the fragment library:

    python scripts/make_hydra_config.py -i config.stub.txt
    sample1	/full/path/to/file/sample1.pos.bam	374.23	12	3
    sample2	/full/path/to/file/sample2.pos.bam	398.19	20	3
    sample3	/full/path/to/file/sample3.pos.bam	401.78	23	3
	
Just redirect the output to a new, complete config file and you should be
ready to go:

    python scripts/make_hydra_config.py -i config.stub.txt > config.hydra.txt


###2. Extract discordant alignments.
=================================
Once you have created a configuration file for Hydra-Multi, you need to run the
`extract_discordants.py` script to, you guessed it, extract the discordant 
alignments from your BAM files into BEDPE format for HydaMulti.

For each inout BAM file listed in your configuration file, 
`extract_discordants.py` will create a BEDPE file of the discordant alignments
in the the same directory.  For example, it will create a `sample1.pos.bam.bedpe` 
file for the `sample1.pos.bam` input file listed in the config file:

    python scripts/extract_discordants.py -c config.hydra.txt -d <sample_name>


###3. Run HydraRouter
=================================
This routes all of the alignments on with the same chromosome/orientation set to the same file for assembly.

    $ hydra-router -config config.hydra.txt -routedList routed-files.txt


###4. Assemble SV breakpoint clusters
==================================
Assembly of each chromosome/orientation set.

    $ sh scripts/assemble-routed-files.sh config.hydra.txt routed-files-test.txt <nb of threads> <punt parameter>

Punt parameter: set to 5x the average coverage of the sample(s)
In the case of one MP sample, ~40x, this lasts ~10days...

###5. Combine the individual SV assembly files into a single file.
===============================================================
Combine all of the chromosome/orientation sets back into one file.

    $ sh scripts/combine-assembled-files.sh /full/path/to/assembled/files/ all.assembled

Comments:

- /path/to/assembled/files, not /path/to/assembled/files/ ! 
- Note all assembly files (punted & assembled) will be deleted, cp to a backup folder or edit code accordingly.

###6. Finalize the SV breakpoint predictions.
===============================================================

    $ scripts/forceOneClusterPerPairMem.py -i all.assembled -o all.sv-calls -m 8G

-This involves several rounds of heavy file sorting, memory allocation (-m, default is 2G) shall be set accordingly.

-This step outputs several files containing the calls: .final, .detail, .all ; that are formatted as follows 
(from https://code.google.com/archive/p/hydra-sv/wikis/FileFormats.wiki )


- Hydra breakpoint output file format (.final and .all)

$1. chrom1 Chromosome for end 1 of the breakpoint. 

$2. start1 Start position for end 1 of the breakpoint. 

$3. end1 End position for end 1 of the breakpoint. 

$4. chrom2 Chromosome for end 2 of the breakpoint. 

$5. start2 Start position for end 2 of the breakpoint. 

$6. end2 End position for end 2 of the breakpoint. 

$7. breakpointId Unique Hydra breakpoint identifier. 

$8. numDistinctPairs Number of distinct pairs in breakpoint. 

$9. strand1 Orientation for the first end of the breakpoint. 

$10. strand2 Orientation for the second end of the breakpoint. 

$11. meanEditDist1 Mean edit distance observed in end1 of the breakpoint pairs. 

$12. meanEditDist2 Mean edit distance observed in end2 of the breakpoint pairs. 

$13. meanMappings1 Mean number of mappings for end1 of all pairs in the breakpoint. 

$14. meanMappings2 Mean number of mappings for end2 of all pairs in the breakpoint. 

$15. breakpointSize Size of the breakpoint. 

$16. numMappings Total number of mappings included in the breakpoint. 

$17. allWeightedSupport Amount of weighted support from the mappings in the breakpoint. 

$18. finalSupport Amount of final support from the mappings in the breakpoint. 

$19. finalWeightedSupport Amount of final weighted support from the mappings in the breakpoint. 

$20. numUniquePairs Number of pairs in the breakpoint that were uniquely mapped to the genome. 

$21. numAnchoredPairs Number of pairs in the breakpoint that were mapped to the genome in an "anchored" fashion (i.e. 1xN). 

$22. numMultiplyMappedPairs Number of pairs in the breakpoint that were multiply mapped to the genome in fashion (i.e. NxN).


- Hydra breakpoint output detail file format

$1. chrom1 Chromosome for end 1 

$2. start1 Start position for end 1 

$3. end1 End position for end 1 

$4. chrom2 Chromosome for end 1 

$5. start2 Start position for end 2 

$6. end1 End position for end 2 

$7. name Name of the pair 

$8. mate1End To which mate of the pair do fields 1,2,3,9,11 correspond? (values: 1 or 2) 
$9. strand1 Orientation for end 1 (+ or -) 

$10. strand2 Orientation for end 2 (+ or -) 

$11. editDist1 Alignment edit distance for end 1 (can be extracted from NM tag in SAM) 

$12. editDist2 Alignment edit distance for end 2 (can be extracted from NM tag in SAM) 

$13. numMappings1 Number of mappings for end 1 of this pair 

$14. numMappings2 Number of mappings for end 2 of this pair 

$15. mappingType What type of mapping is this? (1=unique, 2=anchored, 3-multiply) 

$16. includedInBreakpoint Was this pair ultimately included in this breakpoint? 

$17. breakpointId Unique Hydra breakpoint identifier. 

###7. Determine presence of the SV breakpoint predictions in samples.
=======================================================================

    $ scripts/frequency.py -f all.sv-calls.final -d all.sv-calls.detail -c config.hydra.txt > all.sv-calls.freq

Note also the -x option / do not print column headers    
    
###8. Change footprint intervals into breakpoint intervals.
===============================================================

    $ scripts/hydraToBreakpoint -i all.sv-calls.freq >  all.sv-calls.bkpts
    
