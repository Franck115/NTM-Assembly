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

