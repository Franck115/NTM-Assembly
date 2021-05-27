# NTM-Assembly

Hi,
Franck here, 
This is the assembly script using the trycycler assembler

# installation
## installing with conda
## Installing trycycler
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

## Installing filtlong
```bash 
conda install -c bioconda filtlong 
```

# filtering 

```bash
filtlong --min_length 1000 --keep_percent 95 ccs-reads > reads.fastq
```


# Assembly using trycycler

## Step 1
Creating multiple reads subsets
```bash
trycycler subsample --reads reads.fastq --out_dir read_subsets
```
## Assembly of the subsets using 3 diff assemblers and moving the ouput to a new folder
mkdir assemblies

## Assembly using flye
```bash
flye --pacbio-hifi read_subsets/sample_01.fastq --threads 16  --plasmids --out-dir assembly_01
cp assembly_01/assembly.fasta assemblies/assembly_01.fasta && rm -r assembly_01
```

## Assembly using miniasm_minipolish 
```bash
minimap2 -t 8 -x ava-ont read_subsets/sample_02.fastq read_subsets/sample_02.fastq > overlaps.paf
miniasm -f read_subsets/sample_02.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_02.fastq assembly.gfa > assembly_02.gfa
```
## Assembly using Hifiasm on 4 subset
```bash
hifiasm -o ./hifiasm/assembly_03.fasta -t 32 sample_03.fastq
gfatools gfa2fa assembly_03.fasta.p_ctg.gfa >assembly_03.fasta
```
# Step 2 
Trycycler cluster to group similar contigs
run trycycler cluster to group similar contigs
```bash
trycycler cluster --assemblies assemblies/*.fasta --reads reads.fastq --out_dir trycycler
```

# Step 3
inspect the cluster to decide which are good,rename or delete the bad clusters, and reconcile the good clusters
#lets take cluster_001 is good
trycycler reconcile --reads reads.fastq --cluster_dir trycycler/cluster_001

# Step 4
run a multiple sequence alignment on the cluster
trycycler msa --cluster_dir trycycler/cluster_001

#run trycycler partition to partition the reads
trycycler partition --reads reads.fastq --cluster_dir trycycler/cluster_*

#run Trycycler consensus to make a consensus sequence for each contig cluster
trycycler consensus --cluster_dir trycycler/cluster_001

#to Combine all consensus sequences into a single FASTA
cat trycycler/cluster_*/7_final_consensus.fasta > assembly.fasta

# Quality assessment of the assemblies
```bash
checkm lineage_wf --pplacer_threads 8 -t 8 -x fasta path-to-inputfile path-to-outputfile
```
change the 
# Quality assessmenlt using metrics like N50
seqkit stats inputfile



