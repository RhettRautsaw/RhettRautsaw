# HiFi Genome Assembly 

**Rhett Rautsaw**\
**PacBio, Field Applicaitons Bioinformatic Support (FABS)**

## Setup Directory & Tools
Many of the tools I use in this demo are already available on with nearly any Linux system. However, there are a few additional tools we need to download. 
- To process PacBio data including indexing bam files and converting to fastq, we will use [pbtk](https://github.com/PacificBiosciences/pbtk). 
- To assemble the genome, we will use the latest release of [hifiasm](https://github.com/chhylp123/hifiasm). 
- To calculate simple assembly statistics such as N50, we will use [SeqKit](https://github.com/shenwei356/seqkit).
```
mkdir /path/to/working/directory
mkdir -p bin
cd bin

wget https://github.com/PacificBiosciences/pbtk/releases/download/v3.5.0/pbtk.tar.gz
mkdir pbtk; tar xvzf pbtk.tar.gz -C pbtk; rm pbtk.tar.gz

wget https://github.com/chhylp123/hifiasm/archive/refs/tags/0.24.0.tar.gz
tar xvzf 0.24.0.tar.gz; rm 0.24.0.tar.gz; mv hifiasm-0.24.0 hifiasm; cd hifiasm; make

cd ..

wget https://github.com/shenwei356/seqkit/releases/download/v2.9.0/seqkit_linux_amd64.tar.gz
tar xvzf seqkit_linux_amd64.tar.gz; rm seqkit_linux_amd64.tar.gz

cd ..
```

## Download Data
Demo datasets for all applications can be found on [PacBio's website](https://www.pacb.com/connect/datasets/). I am using the *Helicobacter pylori* dataset generated on the Sequel II ([sample bc2009](https://downloads.pacbcloud.com/public/dataset/2021-11-Microbial-96plex/demultiplexed-reads/)) to demonstrate genome assembly because this species has a relatively small genome size (1 chromosome = 1.6 Mbp) and should assemble relatively quickly. 

This dataset is ~54 Mbp of HiFi data which is equivalent to ~34x coverage of the genome. Note that this is only 0.06% of Revio's potential output (~90 Gbp). If you used all 384 of our SMRTbell barcoded adapters to multiplex this species on a Revio SMRT Cell, you would still only be using 23-25% of a Revio SMRT Cell.

<br>
<br>

```
wget https://downloads.pacbcloud.com/public/dataset/2021-11-Microbial-96plex/demultiplexed-reads/m64004_210929_143746.bc2009.bam
```

## Index and Convert to Fastq
We just downloaded an unaligned BAM (uBAM) file – which is the standard format for PacBio sequencing and what is output from our instruments like Revio and Sequel IIe. We utilize uBAM rather than fastq because BAM format remains an industry standard that can be easily manipulated by existing tools like samtools. This format allows for high compression of the data to keep file sizes small, but adds flexibility to store data beyond basepairs and base qualities – such as location and probability of methylation on reads. 

The files coming off our instruments will come with a supplementary pbi index file to allow for faster data parsing; however, the data we just downloaded does not have this index file. Additionally, we will need to convert the data to fastq format for hifiasm. 

Between these two steps, I am creating symbolic links to keep the original file names (`m64004_210929_143746.bc2009`), but allowing me to reference them through an easier to remember name (`bacteria.hifi_reads`). These steps are optional.

Finally, I'm gzipping (compressing) the fastq file using pigz. 
```
bin/pbtk/pbindex m64004_210929_143746.bc2009.bam

ln -s m64004_210929_143746.bc2009.bam bacteria.hifi_reads.bam
ln -s m64004_210929_143746.bc2009.bam.pbi bacteria.hifi_reads.bam.pbi

bin/pbtk/bam2fastq -j 8 -o bacteria.hifi_reads -u bacteria.hifi_reads.bam

pigz -9 -p 8 bacteria.hifi_reads.fastq
```

## Run hifiasm
After that, we are ready to run hifiasm and it couldn't be easier. A single line of code will do the trick. This will take approximately 35-40 seconds to complete and peak at 16-20 GB of RAM. 
```
bin/hifiasm/hifiasm -t 8 -o bacteria.asm bacteria.hifi_reads.fastq.gz
```

## Convert GFA to FASTA
hifiasm outputs data in GFA format; however, we typically want our assemblies in FASTA format. You can easily make the conversion with this simple awk command. 

Note that because we did not include any supplemental data (trio or Hi-C), only a primary assembly was generated. If additional data was included, a separate file for each haplotype would be generated. 
```
awk '/^S/{print ">"$2;print $3}' bacteria.asm.bp.p_ctg.gfa > bacteria.asm.bp.p_ctg.fasta
```

## Examine Contig Lengths and Calculate Stats
Finally, let's just take a look at the distribution of contig lengths and calculate some summary statistics with seqkit. 
```
samtools faidx bacteria.asm.bp.p_ctg.fasta
cut -f1,2 bacteria.asm.bp.p_ctg.fasta.fai | sort -nr -k2,2

bin/seqkit stats -j 8 -a bacteria.asm.bp.p_ctg.fasta
```

|num_seqs|sum_len  |N50      |N50_num|GC(%)|
|:-------|:--------|:--------|:------|:----|
|1       |1,645,757|1,645,757|1      |39.18|

## Results
You'll note that only one contig is assembled and the contig name ends with "c" indicating it is a circular assembly. Meaning the full chromosome was assembled! 

Now this was a sample that was cultured in a petri dish in the lab. Not all assemblies will look this clean, so don't be surprised if you assemble a lot of contigs especially in Eukaryotes and other more complex genomes. This is due to a few reasons:
1. There are many more chromosomes to assemble...
2. Centromeres are difficult to assemble across; therefore, you will likely have two contigs per chromosome (one for each arm of the chromosome).
	- Note that while centromeres are challenging, they are not impossible with higher coverage and supplementary data to span the region.
3. You'll may also see an abundance of contigs <20-100kb in size. While one of these may be the mitochondrial genome, it is likely that the majority are bacterial and other contaminants that could be filtered out from the assembly. 

# THE END
