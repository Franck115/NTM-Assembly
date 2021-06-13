This example will guide you through the steps involved in the assembly of a genome using the trycycler tool.

for this example, we will use the read "ccs-1968-01179-m64105_201212_035942.Q20.fastq.gz"
make sure it is present in your path and all necessary tools has been installed.

activate your conda environment with 
```bash 
conda activate trycycler 
```
## Filtering the reads
```bash 
filtlong --min_length 1000 --keep_percent 95 ccs-1968-01179-m64105_201212_035942.Q20.fastq.gz > reads.fastq
```
this should produce a new file called "reads.fastq" in your direcctory

## generating assemblies 

```bash
trycycler subsample --reads reads.fastq --out_dir read_subsets
```
![image](https://user-images.githubusercontent.com/84844757/121793418-05d6e300-cbff-11eb-808b-b12f1146506a.png)

### assembly using Flye

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

 Step 2 
run trycycler cluster to group similar contigs
```bash
trycycler cluster --assemblies assemblies/*.fasta --reads reads.fastq --out_dir trycycler
```
## Output



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
