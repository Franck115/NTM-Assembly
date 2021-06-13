This example will guide you through the steps involved in the assembly of a genome using the trycycler tool.

for this example, we will use the read "ccs-1968-01179-m64105_201212_035942.Q20.fastq.gz"
make sure it is present in your directory and all necessary tools has been installed.
Read the readme file to see the tools needed and their use

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
sample_01.fasta,sample_04.fasta, sample_07.fasta, and sample_10.fasta will both be assembled using flye and the output moved to the directory named "assemblies"
```bash
mkdir assemblies
flye --pacbio-hifi read_subsets/sample_01.fastq --threads 16  --plasmids --out-dir assembly_01
cp assembly_01/assembly.fasta assemblies/assembly_01.fasta && rm -r assembly_01
flye --pacbio-hifi read_subsets/sample_04.fastq --threads 16  --plasmids --out-dir assembly_04
cp assembly_04/assembly.fasta assemblies/assembly_04.fasta && rm -r assembly_04
flye --pacbio-hifi read_subsets/sample_07.fastq --threads 16  --plasmids --out-dir assembly_07
cp assembly_07/assembly.fasta assemblies/assembly_07.fasta && rm -r assembly_07
flye --pacbio-hifi read_subsets/sample_10.fastq --threads 16  --plasmids --out-dir assembly_10
cp assembly_10/assembly.fasta assemblies/assembly_10.fasta && rm -r assembly_10
```
## Assembly using miniasm_minipolish 

```bash
minimap2 -t 8 -x ava-ont read_subsets/sample_02.fastq read_subsets/sample_02.fastq > overlaps.paf
miniasm -f read_subsets/sample_02.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_02.fastq assembly.gfa > assembly_02.gfa
any2fasta assembly_02.gfa > assemblies/assembly_02.fasta && rm assembly_02.gfa overlaps.paf assembly.gfa
minimap2 -t 8 -x ava-ont read_subsets/sample_05.fastq read_subsets/sample_05.fastq > overlaps.paf
miniasm -f read_subsets/sample_05.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_05.fastq assembly.gfa > assembly_05.gfa
any2fasta assembly_05.gfa > assemblies/assembly_05.fasta && rm assembly_05.gfa overlaps.paf assembly.gfa
minimap2 -t 8 -x ava-ont read_subsets/sample_08.fastq read_subsets/sample_08.fastq > overlaps.paf
miniasm -f read_subsets/sample_08.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_08.fastq assembly.gfa > assembly_08.gfa
any2fasta assembly_08.gfa > assemblies/assembly_08.fasta && rm assembly_08.gfa overlaps.paf assembly.gfa
minimap2 -t 8 -x ava-ont read_subsets/sample_11.fastq read_subsets/sample_11.fastq > overlaps.paf
miniasm -f read_subsets/sample_11.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_11.fastq assembly.gfa > assembly_11.gfa
any2fasta assembly_11.gfa > assemblies/assembly_11.fasta && rm assembly_11.gfa overlaps.paf assembly.gfa
```
## Assembly using Hifiasm 
Hifiasm assembler will be used to assemble the remaing samples, that is sample_03,06,09 and 12
The codes below  will create a directory for the assembly output, assemble the sample using hifiasm,   change the output of the hifiasm from gfa into fasta format using the gfatool and lastly, copy the output to the assemblies directory 

```bash
mkdir hifiasm
hifiasm -o ./hifiasm/assembly_03.fasta -t 32 read_subsets/sample_03.fastq
gfatools gfa2fa hifiasm/assembly_03.fasta.p_ctg.gfa >assembly_03.fasta
cp assembly_03.fasta assemblies/assembly_03.fasta 
hifiasm -o ./hifiasm/assembly_06.fasta -t 32 read_subsets/sample_06.fastq
gfatools gfa2fa hifiasm/assembly_06.fasta.p_ctg.gfa >assembly_06.fasta
cp assembly_06.fasta assemblies/assembly_06.fasta
hifiasm -o ./hifiasm/assembly_09.fasta -t 32 read_subsets/sample_09.fastq
gfatools gfa2fa assembly_09.fasta.p_ctg.gfa >assembly_09.fasta
cp assembly_09.fasta assemblies/assembly_09.fasta
hifiasm -o ./hifiasm/assembly_12.fasta -t 32 read_subsets/sample_12.fastq
gfatools gfa2fa assembly_12.fasta.p_ctg.gfa >assembly_12.fasta
cp assembly_12.fasta assemblies/assembly_12.fasta
```

 # Step 2 grouping similar contigs 
run trycycler cluster to group similar contigs together
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
