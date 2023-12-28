# Trycycler hybrid assembler - Summary

***

## Create and prepare sequencing

### Short read assemblers for later polishing step

- **Spades: <https://github.com/ablab/spades>**
- **SKESA: <https://github.com/ncbi/SKESA>**
- **Megahit: <https://github.com/voutcn/megahit>**

### Long read assembly

#### Initial run QC - check run parameters

- NanoPlot: <https://github.com/wdecoster/NanoPlot>
- minion_qc: <https://github.com/roblanf/minion_qc>

#### Read QC before assembly

- FastQC: <https://www.bioinformatics.babraham.ac.uk/projects/fastqc/>
- MultiQC: <https://github.com/ewels/MultiQC>
- Blobtools2/Blobtoolkit: (check for contaminating reads) <https://github.com/blobtoolkit/blobtoolkit>

***

## ONT Basecalling using guppy

- Run guppy on a single barcode

```bash
# input fast5 data from ONT sequencers
input_path="path/to/input/fast5/folder"
output_path="path/to/output/folder"
# Select the approriate config model
configModel="dna_r9.4.1_450bps_sup.cfg"
# Run guppy
guppy_basecaller -i $input_path -s $output_path -c $configModel -x auto -r --trim_adapters --compress_fastq
```

- For a single barcode - concatenate all fastq outputs

```bash
# Multiple fastq inputs (one barcode)
inputFastqName="path/to/fastq/for/one/barcode/*.fastq.gz"
outputFastqName="path/and/filename/for/concatentated/fastq/output_filename.gastq.gz"
cat $inputFastqName > $outputFastqName
```

***

### Fastq Filtering

- NanoFilt: <https://github.com/wdecoster/nanofilt>
- Chopper (No GC filtering): <https://github.com/wdecoster/chopper>

### Final QC before assembly

- FastQC: <https://www.bioinformatics.babraham.ac.uk/projects/fastqc/>
- MultiQC: <https://github.com/ewels/MultiQC>
- Blobtools2/Blobtoolkit: <https://github.com/blobtoolkit/blobtoolkit>

***

## Long ONT read assembly

- **Using the Trycycler workflow: <https://github.com/rrwick/Trycycler>**
- **Partial automation using selected scripts: <https://github.com/bogemad/trycycler_scripts>**
- **In the Conda Trycycler environment all assemblers to be used must be installed**
(flye, miniasm, raven, necat, nextdenovo)

### 1. Run the high_coverage_assemly_script
<https://github.com/bogemad/trycycler_scripts/blob/main/high_cov_assembly_script.sh>**

```bash
# Arguents needed by the script
pathToScript="path/to/high_cov_assembly_script.sh"
inputDir=path/to/parent/directory/of/reads
output_fasta=path/to/output/fasta
genome_size=4500000
threads=25
assembly_out_dir=path/to/parent/assembly/dir
 # Command to run the script
bash $pathToScript $sample_name $output_fasta $genome_size $threads $assembly_out_dir
```

- *To turn ignore one or more assemblers used in the **high_cov_assembly_script.sh** open the script and comment out the unwanted assembler/s*

### 2. Reconcile assemblies (Manual checking and selection)

This is the limiting step as each assembly must be reconciled manually. Meaning, find all the clusters for each barcode that can be reconciled and then run the script on each cluster that pass the guidelines summary (below).

<https://github.com/bogemad/trycycler_scripts/blob/main/exnrec.sh>**

Check multiple assembler outputs from "user_filename.trycycler.log.txt" output

- Output from each assembler will have the following letter prefix in trycycler ABC = flye, DEF = miniasm, GHI = raven, JKL = necat, MNO = nextdenovo
- Follow the quality guidelines from the trycycler github:
<https://github.com/rrwick/Trycycler/wiki/Clustering-contigs>
- Guidlines summary:
- 15x minimum coverage in cluster
- At least 4 contigs have relatively consistent size and coverage (exception: some assemblers will sometimes produce a double plasmid - these can be ID'd as they will have 1/2 the coverage, these won't make the cluster invalid but need to remove or edit doubles for reconcile)
- Consider assembly method invalid if no chomosome near expected size (for example if DEF (miniasm) have maximum contig size of 60% the expected genome size then consider assembler output invalid for below rule)
- Indel number of < 400

```bash
# Inputs
path_to_cluster_directory=path/to/the/cluster/directory
reads=file_name.fastq
threads=24 #Set number of threads the script will use
#For the following, contigs "a" (flye assembly) and "d" (miniasm assembly) will be moved to another directory and ignored
contig_letters_to_exclude="ad"
# Run the script
exnrec.sh ${path_to_cluster_directory} ${reads} ${threads} ${contig_letters_to_exclude}
```

### 3. Multiple sequence alignment
<https://github.com/rrwick/Trycycler/wiki/Multiple-sequence-alignment>**

- All the sequences that have passed reconcile (step 2) from each cluster must be aligned.
  - Rarely, this script will fail but it can be run independently using the MUSCLE multiple sequencing aligner.
  - If run independently then change the output name to "3_msa.fasta"

```bash
# Input
pathToCluster="path/to/sequence/cluster/to/be/aligned"
# Run the command
trycycler msa --cluster_dir $pathToCluster
```

### 4. Trycycler partition
<https://github.com/rrwick/Trycycler/wiki/Partitioning-reads>**

```bash
# Input the initial read file
fastqFile="path/to/the/fastq/file/for/barcode/of/interest/filename.fastq.gz"
# Set the paths to the 
clusterDirs="path/to/cluterDir1 path/to/cluterDir2 path/to/cluterDir3"
trycycler partition --reads $fastqFile --cluster_dirs $clusterDirs
```

### 5. Generate a consensus
<https://github.com/rrwick/Trycycler/wiki/Generating-a-consensus>**

- Run the consensus script separately for each good cluster

```bash
trycycler consensus --cluster_dir trycycler/cluster_001
```

### 6. Polishing after Trycycler
<https://github.com/rrwick/Trycycler/wiki/Polishing-after-Trycycler>**

- The Medaka assembly polisher is used
  - <https://github.com/nanoporetech/medaka>

```bash
# Input the initial read file
basecallsFilepath="path/to/the/fastq/file/for/barcode/of/interest/filename.fastq.gz"
draftAssembly="path/to/draft/assembly/7_final_consensus_assembly_name.fasta"
outputDirectory="path/to/the/directory/holding/polished/assembly"
NumCores=32
# Select the appropriate model (high accuracy model used in this example)
modelSettings=r941_min_hac_g507
medaka_consensus -i ${basecallsFilepath} -d ${draftAssembly} -o ${outputDirectory} -t ${NumCores} -m ${modelSettings}
```

