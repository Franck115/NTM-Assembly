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
To filter the reads with fitlong, run the following code

```bash
filtlong --min_length 1000 --keep_percent 95 path/to/the/ccs-reads > reads.fastq
```
this should produce a new file called "reads.fastq" in your direcctory

# Step 1
The first step of trycycler consist of generating multiple assemblies.
firstly, the quality control reads are subsamsample into different read set using the trycycler subsample command:

```bash
trycycler subsample --reads reads.fastq --out_dir read_subsets
```
when done, the reads subsets will be found under the directory "read_subsets"

The second step consist of assembling the read subsets with the various assemblers and moving the output into    new folder named assemblies.

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
# Step 2 
run trycycler cluster to group similar contigs
```bash
trycycler cluster --assemblies assemblies/*.fasta --reads reads.fastq --out_dir trycycler
```
## Output
contigs.phylip:a matrix of the Mash distances between all contigs in PHYLIP format 

contigs.newick: a [FastME](https://academic.oup.com/mbe/article/32/10/2798/1212138) tree of the contigs built from the distance matrix. This can be visualised in a phylogenetic tree viewer such as [FigTree](http://tree.bio.ed.ac.uk/software/figtree/) or [Dendroscope](http://dendroscope.org/) 

# Step 3
inspect the cluster to decide which are good,rename or delete the bad clusters, and reconcile the good clusters
lets take cluster_001 
```bash
trycycler reconcile --reads reads.fastq --cluster_dir trycycler/cluster_001
```

# Step 4
run trycycler msa for each good cluster
```bash
trycycler msa --cluster_dir trycycler/cluster_001
```

# Step 5
run trycycler partition to partition the reads
the * can be used to glob all the good clusters
```bash
trycycler partition --reads reads.fastq --cluster_dir trycycler/cluster_*
```
# Step 6
run Trycycler consensus to make a consensus sequence for each contig cluster
```bash
trycycler consensus --cluster_dir trycycler/cluster_001
```
# Step 7
Combine all consensus sequences into a single FASTA
```bash
cat trycycler/cluster_*/7_final_consensus.fasta > assembly.fasta
```

# Quality assessment of the assemblies
```bash
checkm lineage_wf --pplacer_threads 8 -t 8 -x fasta path-to-inputfile path-to-outputfile
```




