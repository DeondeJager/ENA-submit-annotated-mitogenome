# Submit annotated mitogenome to ENA
This repository serves mainly as a future reference for myself for how I submitted an assembled mitogenome (n = 1) to the ENA. 
But maybe it will be useful to others.

SciLifeLab has good tutorials on ENA submissions [here](https://data-guidelines.scilifelab.se/topics/ena-submission-tutorial/), which I often check.

>[!warning]
>As of 22 September 2026, the submission described below still has a "failed" submission status since being submitted on 20 Sep.
>The error message is "The post-archival processing of the file has failed".
>
>However, the ENA states that failed submissions are retried automatically and a ticket should only be logged if a week has passed without success.

## 1. Register the locus_tag *prefix*
[Link](https://ena-docs.readthedocs.io/en/latest/submit/assembly/genome.html#register-locus-tag-prefixes) to relevant ENA docs page.

>[!note]
>This assumes you have already registered your [study/project](https://ena-docs.readthedocs.io/en/latest/submit/study.html) and [sample](https://ena-docs.readthedocs.io/en/latest/submit/samples.html) in the ENA.

1. On the Webin Submissions portal front page, click on `Studies Report`, then click the box with arrow under the `Actions` field on the right of the project into which you want to upload the mitogenome. 
2. In the project editor, tick the box that says "Will you provide functional genome annotation?"
2. Add a locus tage prefix in the `Locus Tag Prefix Registration` box that appears at the bottom of the page.

>[!note]
>It is just the *prefix* that needs to be registered, not the actual locus tags.

A locus tag prefix is a globally unique codeword that will be used when we convert the annotations `gff` to `embl` format.

I used "FOSSIL936".

This gets checked against the list of existing locus tags in the INSDC database to make sure no-one else has used it.
This process takes 24 hours, so has to be done before you want to submit your mitogenome.

It is a way to assign a generic unique code to the genes annotated in your genome specifically.

In the `embl` file, the program `EMBLmyGFF3` (see below) will then add `_LOCUS01` to the `locus tag prefix` to give `FOSSIL936_LOCUS01` for first annotation and increase the number sequentially from there. 

The number of zeros used for padding is automatically chosen depending on the number of genes in your gff file.

## 2. Annotate assembled mitogenome
The final genome was annotated from the blue wildebeest reference (JN632628.1) using the `Transfer Annotation` function in Geneious with an 80% identity cut-off.

I then did the following:
1. Exported the mitogenome as a fasta file and changed the sequence name in Notepad++ to "Fossil936". 
2. Exported the annotations in gff3 format `GFF3 annotations (*.gff)` without the sequence and **not** in strict format, so "Export all qualifiers...", as the strict mode removed the `Product=` information from the 8th column and I wanted to keep this info as it gives the tRNA, rRNA and protein coding gene names.
3. Manually edited the gff file to:
  - 3.1. Change all instances of the sequence name from whatever is was in Geneious to "Fossil936" using find-and-replace.
  - 3.2. Assign each `tRNA`, `rRNA`, and `CDS` feature (these are children features of genes) to a parent `gene` using the `Parent=<insert gene ID>` terminology in the 8th column. 
  I  made sure the parent `gene` feature was always listed before the child feature in the gff (not sure if necessary though).
  - 3.3. Removed any `transl_except` flags from the 8th column of `CDS` features as they were causing validation errors (I think as a consequence of being transferred from another mitogenome).
  - 3.4. Changed the annotation `sequence_feature` for the control region to `misc_feature`, as the former is no longer an accepted term for annotation features (see [here](https://www.ebi.ac.uk/ena/WebFeat/)).

Point 3.2. is particularly important with regards to the locus tags, as both the parent and child annotations must have the same `/locus_tag`, as they refer to the same feature/locus/gene. 

Without assigning parent-child relationships, each `gene` and `CDS` feature, for example, would get a different `/locus_tag` in the `embl` file, which would be incorrect.

This was only a problem for me because I originally downloaded and imported the blue wildebeest mitogenome into Geneious as a GenBank (`.gb`) format file, which does not encode the `Parent` information.
However, the `gff` format does, so when downloading from NCBI (or similar), it is best to download the sequence in `fasta` format and the annotations in `gff` format for importing into Geneious, as this preserves the `Parent` information. 

## 3. Generate embl format of annotations
Convert the edited `gff` to `embl` format.

See the `embl` flatfile format manual [here](https://raw.githubusercontent.com/enasequence/read_docs/master/submit/fileprep/flatfile_user_manual.txt) and the `gff` format specification [here](https://github.com/The-Sequence-Ontology/Specifications/blob/master/gff3.md).

1. Install `EMBLmyGFF` (GitHub [page](https://github.com/NBISweden/EMBLmyGFF3) with installation instructions).

2. Run it
```
EMBLmyGFF3 --data_class WGS --organelle mitochondrion --locus_tag FOSSIL936 --locus_zero_padding --molecule_type "genomic DNA" --project_id PRJEB108657 --transl_table 2 --species "Connochaetes sp. DdJ-2026" --topology circular --author "list all authors" --output Fossil936.embl Fossil936_relax_manual.gff Fossil936.fasta
```
Check that the output file is what you expect.

3. Compress it
The Webin-CLI program expects either a `.gz` or `.bzip2` compressed file.

`gzip -c Fossil936.embl > Fossil936.embl.gz`

I use the `-c` flag so I keep the uncompressed version for easy viewing and manual editing if needed.

## 4. Generate chromosome list file
[Link](https://ena-docs.readthedocs.io/en/latest/submit/fileprep/assembly.html#chromosome-list-file) to relevant ENA docs page.

This is a tab-separated file with four fields:
1. OBJECT_NAME or sequence name: Has to match the name in the `AC * ` line of the `embl` file, so in my case "_Fossil936".
2. CHROMOSOME_NAME: MT for mitochondrion
3. CHROMOSOME_TYPE: circular-chromosome for mitochondrion
4. CHROMOSOME_LOCATION: Mitochondrion

Make it in a text editor; mine looked like this:

`_Fossil936	MT	circular-chromosome	Mitochondrion`

This file also needs to be compressed like the `embl` file:

`gzip -c chr_list.tsv > chr_list.tsv.gz`

## 5. Make manifest file
[Link](https://ena-docs.readthedocs.io/en/latest/submit/assembly/genome.html#stage-2-prepare-the-files) to relevant ENA docs page.

Finally, a manifest file is also required, which contains some sample and sequencing metadata and the names of the `embl` and `chromosome list` files.

Here is a template with all possible fields for a *genome assembly* (see `manifest_template.txt`):
```
# The following metadata fields are supported in the manifest file for genome context:
STUDY: Study accession - mandatory
SAMPLE: Sample accession - mandatory
ASSEMBLYNAME: Unique assembly name, user-provided - mandatory
ASSEMBLY_TYPE: ‘clone or isolate’ - mandatory
COVERAGE: The estimated depth of sequencing coverage - mandatory
PROGRAM: The assembly program - mandatory
PLATFORM: The sequencing platform, or comma-separated list of platforms - mandatory
MINGAPLENGTH: Minimum length of consecutive Ns to be considered a gap - optional
MOLECULETYPE: ‘genomic DNA’, ‘genomic RNA’ or ‘viral cRNA’ - optional
DESCRIPTION: Free text description of the genome assembly - optional
RUN_REF: Comma separated list of run accession(s) - optional

# Various file name fields are supported in the manifest file. Note that all of these are optional, though of course at least one must be provided and some may only be relevant in the presence of other file types. 
FASTA: sequences in fasta format
FLATFILE: sequences in EMBL-Bank flat file format
AGP: sequences in AGP format
CHROMOSOME_LIST: list of chromosomes
UNLOCALISED_LIST: list of unlocalised sequences
```

The actual information I entered for this submission (see `Fossil936_manifest.txt`):
```
STUDY: PRJEB108657
SAMPLE: SAMEA121731856
ASSEMBLYNAME: Fossil936 mitochondrion
ASSEMBLY_TYPE: isolate
COVERAGE: 39
PROGRAM: aITE v3 (https://doi.org/10.1111/2041-210X.13990) and MITObim v1.8 (https://doi.org/10.1093/nar/gkt371)
PLATFORM: ILLUMINA
MOLECULETYPE: genomic DNA
DESCRIPTION: Both aITE and MITObim were used twice independently using the two extant wildebeest species Connochaetes taurinus (blue wildebeest: JN632628.1) and C. gnou (black wildebeest: JN632626.1) mitogenomes as bait reference genomes. For the aITE mapper we used version 3, which utilises relaxed mismatch parameters in BWA (-n 0.001 -o 2) and for MITObim we specified a mismatch parameter of 3. We aligned the four resultant mitogenomes to each other and the two bait reference mitogenomes using MAFFT v7.392 specifying --globalpair and --maxiterate 16, and manually built a final consensus sequence by only considering sites where all four resultant mitogenomes were in agreement. The final genome was annotated from the blue wildebeest reference using the Transfer Annotation function in Geneious with an 80% identity cut-off.
RUN_REF: ERR16741334,ERR16741336
FLATFILE: Fossil936.embl.gz
CHROMOSOME_LIST: chr_list.tsv.gz
```
## 6. Validate and submit with Webin-CLI
Download the `.jar` file of latest version of the Webin-CLI java program from [here](https://github.com/enasequence/webin-cli/releases).
1. Put `webin-cli-9.0.3.jar` in working directory (for some reason it would not work for me even when I gave the full path).
2. Validate submission

This checks for errors before actual submission.
```
java.exe -jar webin-cli-9.0.3.jar -context genome -manifest Fossil936_manifest.txt -username Webin-**** -password **** -validate
```
Fix any validation errors that might arise. 

>[!note]
>The real errors are given the `./genome/Fossil936_mitochondrion/validate` folder in a `.report` file.
>Any messages on the screen are just to notify you that there are errors.

3. Submit if validation succeeds
```
java.exe -jar webin-cli-9.0.3.jar -context genome -manifest Fossil936_manifest.txt -username Webin-**** -password **** -submit
```
