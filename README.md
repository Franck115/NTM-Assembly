# NTM-Assembly

Hi,
Franck here, 
This is the assembly script using the trycycler assembler

#first of all we filter the reads by quality using fitlong; using the codes

filtlong --min_length 1000 --keep_percent 95 ccs-reads > reads.fastq

#create multiple reads subsets

trycycler subsample --reads reads.fastq --out_dir read_subsets

#Assembly of the subsets using 3 diff assemblers and moving the ouput to a new folder
mkdir assemblies

#assembly using flye on 4 subset
flye --pacbio-hifi read_subsets/sample_01.fastq --threads 16  --plasmids --out-dir assembly_01
cp assembly_01/assembly.fasta assemblies/assembly_01.fasta && rm -r assembly_01

#assembly using miniasm_minipolish on 4 subset
minimap2 -t 8 -x ava-ont read_subsets/sample_02.fastq read_subsets/sample_02.fastq > overlaps.paf
miniasm -f read_subsets/sample_02.fastq overlaps.paf > assembly.gfa
minipolish -t 8 --pacbio read_subsets/sample_02.fastq assembly.gfa > assembly_02.gfa

#assembly using Hifiasm on 4 subset
hifiasm -o ./hifiasm/assembly_03.fasta -t 32 sample_03.fastq
gfatools gfa2fa assembly_03.fasta.p_ctg.gfa >assembly_03.fasta

#run trycycler cluster to group similar contigs
trycycler cluster --assemblies assemblies/*.fasta --reads reads.fastq --out_dir trycycler

#inspect the cluster to decide which are good,rename or delete the bad clusters, and reconcile the good clusters
#lets take cluster_001 is good
trycycler reconcile --reads reads.fastq --cluster_dir trycycler/cluster_001

#run a multiple sequence alignment on the cluster
trycycler msa --cluster_dir trycycler/cluster_001

#run trycycler partition to partition the reads
trycycler partition --reads reads.fastq --cluster_dir trycycler/cluster_*

#run Trycycler consensus to make a consensus sequence for each contig cluster
trycycler consensus --cluster_dir trycycler/cluster_001

#to Combine all consensus sequences into a single FASTA
cat trycycler/cluster_*/7_final_consensus.fasta > assembly.fasta

#run checkm to assess the quality of the assemblies
checkm lineage_wf --pplacer_threads 8 -t 8 -x fasta path-to-inputfile path-to-outputfile

#Quality assessmenlt using metrics like N50
seqkit stats inputfile



