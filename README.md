MetaMaps
========================================================================

MetaMaps is tool specifically developed for the analysis of long-read (PacBio/Oxford Nanopore) metagenomic datasets.

It simultaenously carries out read assignment and sample composition estimation.

It is faster than classical exact alignment-based approaches, and its output is more information-rich than that of kmer-spectra-based methods. For example, each MetaMaps alignment comes with an approximate alignment location, an estimated alignment identity and a mapping quality.

The approximate mapping algorithm employed by MetaMaps is based on [MashMap](https://github.com/marbl/MashMap). MetaMaps adds a mapping quality model and EM-based estimation of sample composition.

## News
(28 August 2018) We're adding more flexible tools to construct your own databases - see [here](https://github.com/DiltheyLab/MetaMaps/issues/5#issuecomment-414301437) for details. Feedback welcome!

## Installation
Follow [`INSTALL.txt`](INSTALL.txt) to compile and install MetaMaps.

Then download a database, e.g. [miniSeq+H](https://www.dropbox.com/s/6p9o42yufx2vkae/miniSeq%2BH.tar.gz?dl=0) (~8G compressed, microbial genomes and the human reference genome). Extract the downloaded database into the `databases/` directory.

## Usage

Analysis of a dataset with MetaMaps consists of two steps: mapping and classification:

```
./metamaps mapDirectly --all -r databases/miniSeq+H/DB.fa -q input.fastq -o classification_results
./metamaps classify --mappings classification_results --DB databases/miniSeq+H
```

### Memory-efficient mapping

You can use the '--maxmemory' parameter to specify a target for maximum memory consumption (in gigabytes). Note that this feature is implemented heuristically; actual memory usage will fluctuate and execeed the target. We recommend using around 70% of the available memory as a target amount (for example, 20G when you have a 32G machine).

Example:

```
./metamaps mapDirectly --all -r databases/miniSeq+H/DB.fa -q input.fastq -o classification_results --maxmemory 20
./metamaps classify --mappings classification_results --DB databases/miniSeq+H
```

### Multithreading

You can use the parameter `-t` to speed up mapping and classification.

Example:

```
./metamaps mapDirectly -t 5 --all -r databases/miniSeq+H/DB.fa -q input.fastq -o classification_results
./metamaps classify -t 5 --mappings classification_results --DB databases/miniSeq+H
```

Important: if you encounter problems with multithreading efficiency, try `unset MALLOC_ARENA_MAX` immediately before calling MetaMaps.

## Output

MetaMaps outputs both an overall compositional assignment and per-read taxonomic assignments. Specifically, it will (for `-o classification_results`) produce the following files:

1. `classification_results.EM.WIMP`: Sample composition at different taxonomic levels (WIMP = "What's in my pot"). The level "definedGenomes" represents strain-level resolution (i.e., the defined genomes in the classification database). The EM algorithm is carried out at this level.
   
   Output columns: `Absolute` specifies the number of reads assigned (by their maximum likelihood mapping estimate) to the taxonomic entity; `EMFrequency` specifies the estimated frequency of the taxonomic entity prior to taking into account unmapped reads; `PotFrequency` specifies the estimated final frequency of the taxonomic entity (i.e. after correcting for unmapped reads).

2. `classification_results.EM.reads2Taxon`: One line per read, and each line consists of the read ID and the taxon ID of the genome that the read was assigned to. Taxon IDs beginning with an 'x' represent MetaMaps-internal taxon IDs that disambiguate between source genomes attached to the same 'species' node. These can be interpreted using the extended database taxonomy (sub-directory `taxonomy` in the directory of the utilized database).

3. `classification_results.EM.reads2Taxon.krona`: Like `classification_results.EM.reads2Taxon`, but in [Krona](https://github.com/marbl/Krona/wiki) format. Each line has an additional quality value, and only taxon IDs from the standard NCBI taxonomy are used.

4. `classification_results.EM.contigCoverage`: Read coverage for contigs that appear in the final set of maximum-likelihood mappings. Contigs are split into windows of 1000 base pairs. Each line represents one window and specifies the MetaMaps taxonID of the contig (`taxonID`), a species/strain label (`equalCoverageUnitLabel`), the ID of the contig in the classification database FASTA file (`contigID`), start and stop coordinates of the window (`start` and `stop`), the number of read bases overlapping the window (`nBases`), and the number of read bases overlapping the window divided by window length, i.e. coverage (`readCoverage`).

5. `classification_results.EM.lengthAndIdentitiesPerMappingUnit`: Read length and estimated identity for all reads, sorted by the contig they are mapped to. Each line represents one read and has the fields `AnalysisLevel`, which is always equal to `EqualCoverageUnit`; `ID`, which is the ID of the contig in the classification database FASTA file; `readI`, which is a simple numerical read ID; `Identity`, which is the estimated alignment identity; and `Length`, specifying the length of the read.

5. `classification_results.EM`: The final and complete set of approximate read mappings. Each line represents one mapping in a simple space-delimited format. Fields: read ID, read length, beginning of the mapping in the read, end of the mapping in the read, strand, contig ID, contig length, beginning of the mapping in the contig, end of the mapping in the contig, estimated alignment identity using a Poisson model, MinHash intersection size, MinHash union size, estimated alignment identity using a binomial approximation, mapping quality. The mapping qualities of all mappings for one read sum up to 1.

6. `classification_results.EM.evidenceUnknownSpecies`: Various statistics on read identities and zero-coverage regions of identified genomes. These are not particularly useful in their current state; visual inspection of coverage and identity patterns should take precedence.

You can download [example output files](https://github.com/DiltheyLab/MetaMaps/blob/master/MetaMaps_example_output.zip).

## Databases

The 'miniSeq+H' database is a good place to start. It contains >12000 microbial genomes and the human reference genome. We provide miniSeq+H as a download (see above for the link).

You can also download and construct your own reference databases. For example, this is how to construct the miniSeq+H database:

1. Download the genomes you want to include. The easiest way to do this is by copying the RefSeq/Genbank directory structure of the taxonomic branches you're interested in. This can be done with the `downloadRefSeq.pl` script, which is easily customizable (e.g., `--targetBranches archaea,bacteria,fungi` to download these three branches). Example:

    ```
    mkdir download
	perl downloadRefSeq.pl --seqencesOutDirectory download/refseq --taxonomyOutDirectory download/taxonomy
    ```

2. We need to make sure that each contig ID is annotated with a correct and unique taxon ID and we want the whole database as one file. `annotateRefSeqSequencesWithUniqueTaxonIDs.pl` can help:

    ```
    perl annotateRefSeqSequencesWithUniqueTaxonIDs.pl --refSeqDirectory download/refseq --taxonomyInDirectory download/taxonomy --taxonomyOutDirectory download/taxonomy_uniqueIDs
    ```
    
    By default this script will only process complete (finished) assemblies - if you want to modify this behaviour, uncomment the line `next unless($assembly_level eq 'Complete Genome');`.
    
3. We might also manually want to include additional genomes, for example the human reference genome. Obtain the genome in one file (e.g. `hg38.primary.fna`) and add taxon IDs:

    ```
    perl util/addTaxonIDToFasta.pl --inputFA hg38.primary.fna --outputFA hg38.primary.fna.with9606 --taxonID 9606
    ```
4. If the `databases` directory does not exist, create it: `mkdir -p databases`.

5. Finally, construct the MetaMaps databasen (here `myDB`):

    ```
    perl buildDB.pl --DB databases/myDB --FASTAs download/refseq,hg38.primary.fna.with9606 --taxonomy download/taxonomy_uniqueIDs
    ```
	
    The NCBI taxonomy changes on a regular basis, and you might not want to repeat the complete database construction process every time that happens. You can update the utilized taxonomy as part of `buildDB.pl`, by specifying the "old" taxonomy (used for `addTaxonIDToFasta.pl`), `--updateTaxonomy 1`, and the path to a download of the new taxonomy (e.g. [ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz](ftp://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz)). Example:

    ```
    perl buildDB.pl --DB databases/myDB --FASTAs download/refseq/ref.fa,hg38.primary.fna.with9606 --taxonomy download/new_taxonomy --oldTaxonomy download/taxonomy_uniqueIDs --updateTaxonomy 1
    ```
## Advanced features

### Plotting spatial genome coverage and alignment identities

MetaMaps comes with a small R script (`plotIdentities_EM.R`) that helps visalize spatial genome coverage and alignment identity per genome. You can assess these metrics to identify mismatches between the sample and the database.

Parameters: the classification prefix (i.e. whatever input your provided to `metamaps --classify`) of the output.

Example:

```
Rscript plotIdentities_EM.R classification_results
```

### Filtering out WIMP entries with low median identity

If you suspect that your sample contains many genomes not represented in the database (one way to adjudicate this is to examine the identity histograms of the maximum likelihood alignments, e.g. with `plotIdentities_EM.R`), the produced WIMP files may contain many false-positive hits.

You can filter your WIMP output with the script `filterLowIdentityEntities.pl`.

Parameters:

1. `--DB`: Database name.
2. `--mappings`: Path to main MetaMaps mappings file.
3. `--identityThreshold:`: Median identity threshold (between 0 and 1). Default: 0.8, but check the identity histograms to make sure you use a sensible value.

Example:

```
perl filterLowIdentityEntities.pl --DB miniSeq+H --mappings classification_results --identityThreshold 0.8
```

### COG group analysis

You can use MetaMaps to carry out a COG group analysis of your metagenomic sample. Whenever a read overlaps with an annotated gene location, it is counted towards the COG groups (and other annotations) associated with the corresponding gene. Note that a single read can overlap with multiple genes, and that a single gene can be associated with multiple COG groups (or other annotations). Gene locations and amino acid sequences come from GenBank/Refseq, and the annotations are produced with [eggnog-mapper](https://github.com/jhcepas/eggnog-mapper).

1. Download an annotated database.


    To carry out this type of analysis, you need an "annotated" database that contains, in addition to the reference FASTA files, the locations and amino acid sequences of encoded genes, as well as a (gene) -> (annotation) mapping file. The [refseq_with_annotations](https://www.dropbox.com/s/36nz876g1hjh01r/refseq_with_annotations.tar.gz?dl=0) database (18.4 GB compressed) is a good place to start. You can untar this file from the main MetaMaps directory. Afterwards, you should see a `databases/refseq_with_annotations` directory.

2. Carry out mapping and classification.

    Mapping and classification work as before - just make sure to map against the annotated database.

    Example:

    ```
    ./metamaps mapDirectly --all -r databases/refseq_with_annotations/DB.fa -q input.fastq -o classification_results
    ./metamaps classify --mappings classification_results --DB databases/refseq_with_annotations
    ```

3. Carry out the gene- and annotation-level analysis.

    A gene- and annotation-level analysis is carried out with the script `geneLevelAnalysis.pl`. Example:

    ```
    perl geneLevelAnalysis.pl --DB databases/refseq_with_annotations --mappings classification_results
    ```

    This command will produce the following files:
    - `classification_results.EM.geneLevelAnalysis`: This file contains the names and (for some genes) protein IDs of genes hit by overlapping reads from the input dataset. It also contains the number of overlapping reads and their median identity.

    - `classification_results.EM.proteins.*`: Like `classification_results.EM.geneLevelAnalysis`, but agglomerated according to eggnog-mapper-produced gene annotations. For example, `classification_results.EM.proteins.COG` will contain a COG-level analysis of the input data. Note that features are not mutually exclusive, i.e. a single read can overlap with multiple genes, and a single gene can carry multiple annotations.




