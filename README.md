# NTM-Assembly

Hi,
The assembly is performed using the [Trycycler](https://github.com/rrwick/Trycycler) pipeline. 
Trycyler  is an ensemble tool that takes as input multiple separate long-read assemblies from different assemblers (we used the assemblers:[Flye](https://github.com/fenderglass/Flye), [Miniasm](https://github.com/lh3/miniasm)+[Minipolish](https://github.com/rrwick/Minipolish) and [Hifiasm](https://github.com/chhylp123/hifiasm)) to generate consensus long-read assemblies for bacteria.
The pipeline consists of 4 steps: contigs clustering (clustering contig sequences to distinguish complete contigs from incomplete contigs), contigs reconciliation (reconciling alternative contigs against each other and repairing circularization issues), multiple sequence alignment, and consensus sequence construction (constructing a consensus sequence from the MSA).

The entire work and script was run on the ITM server

## installations with conda
The tools used were installed using conda. Most of the tools are already present on the ITM server.
perfforming the work on your local machine will need you to install conda
Conda can be installed through Miniconda. Download and installation information can be found on this [webiste](https://docs.conda.io/en/latest/miniconda.html) 
### Installing trycycler
Trycycler can be installed using the codes below:
creating an environment for trycycler
```bash 
conda create -n trycycler
```

Activate it
```bash 
conda activate trycycler 
```
installing trycycler
```bash 
conda install trycycler
```
You'll need to activate your environment (via conda activate trycycler) before running Trycycler.

### Installing filtlong
[Fitlong](https://github.com/rrwick/Filtlong) will be used for the quality control by filtering the reads by quality in order to produce better subsets.
Fitlong can be installed by conda using the code below:

```bash 
conda install -c bioconda filtlong 
```
### Installing Flye
[Flye](https://github.com/fenderglass/Flye) is a de novo assembler tool for long reads sequencing.
Flye will be one of the three assembler used to generate the assemblies
Flye can be installed by conda using the code below:
```bash 
conda install flye
```
### Installing Miniasm+Minipolish
The second assembler use will be a combinination of miniasm and minipolish.
[Miniasm](https://github.com/lh3/miniasm) is a fast OLC assembler which produced an assembly graph in GFA format as output.
It can be installed by conda using the code below:

```bash 
conda install -c bioconda miniasm 
```
[any2fasta](https://github.com/tseemann/any2fasta) is used to convert ssembly graph in GFA format produced by miniasm into fasta format.
It can be installed by conda using the code below:
```bash 
conda install -c bioconda any2fasta
```
[Minipolish](https://github.com/rrwick/Minipolish) is used to polish the assembly generated
```bash 
conda install -c bioconda minipolish
```
### Installing Hifiasm
[Hifiasm](https://github.com/chhylp123/hifiasm) is the last assembler we will used.
It can be installed by conda using the code below:
```bash 
conda install -c bioconda hifiasm
```
### Installing gfatools
[gfatools](https://github.com/lh3/gfatools) will be used to manipulate and change the gfa format output produced by hifiasm into fasta format
```bash 
conda install -c ohmeta gfatools
```
# Assembly 
After installing all required tools, we can now proceed to the assembling process 

## filtering 
To filter the reads with fitlong, run the following code. this will results in the production of quality reads.

```bash
filtlong --min_length 1000 --keep_percent 95 path/to/the/ccs-reads > reads.fastq
```
this should produce a new file called "reads.fastq" in your direcctory

# Step 1 Generating assemblies
The first step of trycycler consist of generating multiple assemblies.
firstly, the quality control reads are subsamsample into different read set using the trycycler subsample command. this is done in order to have multiple read subsets of the genome independent of each other

```bash
trycycler subsample --reads reads.fastq --out_dir read_subsets
```
when done, the reads subsets will be found under the directory "read_subsets"

The second step consist of assembling the read subsets with the various assemblers in order to produce multiple assemblies independent to each other.

## Assembly using flye
```bash
mkdir assemblies
flye --pacbio-hifi read_subsets/sample_01.fastq --threads 16  --plasmids --out-dir assembly_01
cp assembly_01/assembly.fasta assemblies/assembly_01.fasta && rm -r assembly_01
```

## Assembly using miniasm_minipolish 
```bash
minimap2 -t 8 -x ava-ont read_subsets/sample_02.fastq read_subsets/sample_02.fastq > overlaps.paf
miniasm -f read_subsets/sample_02.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_02.fastq assembly.gfa > assembly_02.gfa
```
## Assembly using Hifiasm 
```bash
hifiasm -o ./hifiasm/assembly_03.fasta -t 32 sample_03.fastq
gfatools gfa2fa assembly_03.fasta.p_ctg.gfa >assembly_03.fasta
```
# Step 2 Clustering contigs

After succesfully producing multiple assemblies,the objective of this step is to cluster the contigs of the assemblies into per-replicon groups and excludes incomplete or misassembled contigs.
```bash
trycycler cluster --assemblies assemblies/*.fasta --reads reads.fastq --out_dir trycycler
```
## Output
contigs.phylip:a matrix of the Mash distances between all contigs in PHYLIP format 

contigs.newick: a [FastME](https://academic.oup.com/mbe/article/32/10/2798/1212138) tree of the contigs built from the distance matrix. This can be visualised in a phylogenetic tree viewer such as [FigTree](http://tree.bio.ed.ac.uk/software/figtree/) or [Dendroscope](http://dendroscope.org/) 
This is done inorder to inspect and decides on which clusters are good or bad. the bad clusters should be removed or renamed.
Directories for each cluster

# Step 3 Reconciling contigs
Trycycler reconcile aims as to make sure the contigs are sufficiently similar to each other,ensure all contig sequences are on the same strand and perform an alignment check. This step is done per good cluster.
lets take cluster_001 
```bash
trycycler reconcile --reads reads.fastq --cluster_dir trycycler/cluster_001
```
## Output
```2_all_seqs.fasta``` is created in the cluster_001 directory
# Step 4 Multiple sequence alignment
A multiple sequence alignment will be run on the reconcilied contig (2_all_seqs.fasta)
The trycycler msa is run per cluster
```bash
trycycler msa --cluster_dir trycycler/cluster_001
```
## Output
```3_msa.fasta``` will be created in the cluster_001 directory

# Step 5 Partitioning reads
This steps aims as to partition the reads between clusters. it is run on the entire genome and not per cluster.
The * can be used to glob all the good clusters
```bash
trycycler partition --reads reads.fastq --cluster_dir trycycler/cluster_*
```
## Output
```4_reads.fastq``` is created in each of the cluster directories.
# Step 6 Generating Consensus
This step consist of generating a consensus contig sequence for each of the good clusters selected. This is done by converting the MSA into a graph form


```bash
trycycler consensus --cluster_dir trycycler/cluster_001
```
## Output
A ```7_final_consensus.fasta``` file is created in each cluster. the file can be combine into a single FASTA file using the following code:

```bash
cat trycycler/cluster_*/7_final_consensus.fasta > assembly.fasta
```
All the assembly produced should be copy into a new directory

# Quality assessment of the assemblies
The quality of the assemblies are then assessed using [CheckM](https://github.com/Ecogenomics/CheckM)

```bash
checkm lineage_wf --pplacer_threads 8 -t 8 -x fasta path-to-inputfile path-to-outputfile
```

# Annotation of the genome
The genome is annotated using [prokka](https://github.com/tseemann/prokka)

The code consist of a loop which will annotate all the genome assemblies in the directory

```bash
for b in *; do prokka "$b" --outdir ./annotation/"$b" -cpus 12 -centre X --compliant; done
```
## Output
![image](https://user-images.githubusercontent.com/84844757/121821757-56038300-cc9b-11eb-84ae-5eb4903a6e71.png)

