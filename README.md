# Unlabeled_HiCzin

## Install and Setup
### conda
We recommend using conda to run HiCzin.

After installing Anaconda (or miniconda), Users can clone the repository with git:
```
git clone https://github.com/dyxstat/HiCzin.git
```

Once complete, you can enter the repository folder and then create a HiCzin environment using conda:
```
# enter the HiCzin folder
cd HiCzin
# Construct environment
conda env create -f HiCzin_linux_env.yaml 
or
conda env create -f HiCzin_osx_env.yaml
# Enter the environment
conda activate HiCzin_env
```

Normalization method in HiCzin depends on R package 'glmmTMB'. Though the R package can be installed by 'conda install -c conda-forge r-glmmtmb', you will meet one potential warning derived from the dependency version (https://github.com/glmmTMB/glmmTMB/issues/615): 'Error in .Call("FreeADFunObject", ptr, PACKAGE = DLL) : "FreeADFunObject" not available for .Call() for package "glmmTMB"' and we are not sure whether this warning would influence the noramlization results. To get rid of this warning, we strongly recommend you to install the source version of package 'glmmTMB' directly in R:

```
# Enter the R
R
# Download the R package and you may need to select a CRAN mirror for the installation
install.packages("glmmTMB", type="source")
```

Finally, you can test the pipeline, and testing result are in test/out/hiczin.log:
```
python ./hiczin.py test test/out
```

## Preparation
### 1.Shotgun assembly
For the shotgun library, de novo metagenome assembly is produced by an assembly software, such as MEGAHIT.
```
megahit -1 SG1.fastq.gz -2 SG2.fastq.gz -o ASSEMBLY --min-contig-len 1000 --k-min 21 --k-max 141 --k-step 12 --merge-level 20,0.95
```
### 2.Align Hi-C paired-end reads to assembled contigs
Hi-C paired-end reads are mapped to assembled contigs using BWA-MEM with parameters ‘-5SP’. Then, samtools with parameters ‘view -F 0x904’ is applied to remove unmapped reads (0x4) and supplementary (0x800) and secondary (0x100) alignments and sort BAM files by name. 
```
bwa index final.contigs.fa
bwa mem -5SP final.contigs.fa hic_read1.fastq.gz hic_read2.fastq.gz > MAP.sam
samtools view -F 0x904 -bS MAP.sam > MAP_UNSORTED.bam
samtools sort -n MAP_UNSORTED.bam -o MAP_SORTED.bam
```

### 3.Calculate the coverage of assembled contigs
Firstly, BBmap from the BBTools suite is applied to map the shotgun reads to the assembled contigs with parameters ‘bamscript=bs.sh; sh bs.sh’. The coverage of contigs is computed using script: ‘jgi summarize bam contig depths’ from MetaBAT2 v2.12.1.
```
bbmap.sh in1=SG1.fastq.gz in2=SG2.fastq.gz ref=final.contigs.fa out=SG_map.sam bamscript=bs.sh; sh bs.sh
jgi_summarize_bam_contig_depths --outputDepth coverage.txt --pairedContigs pair.txt SG_map_sorted.bam
```

## Usage
### Implement the HiCzin normalization
```
python ./hiczin.py norm [Parameters] FASTA_file BAM_file COV_file OUTPUT_directory
```
#### Parameters
```
-e: Case-sensitive enzyme name. Use multiple times for multiple enzymes 
    (Optional; If no enzyme is input, HiCzin_LC mode will be automatically employed to do normalization)
--min-len: Minimum acceptable contig length (default 1000)
--min-mapq: Minimum acceptable alignment quality (default 30)
--min-match: Accepted alignments must be at least N matches (default 30)
--min-signal: Minimum acceptable signal (default 2)
--thres: Maximum acceptable fraction of incorrectly identified valid contacts in spurious contact detection (default 0.05)
-v: Verbose output
```
#### Input File

* **FASTA_file**: a fasta file of the assembled contig (e.g. final.contigs.fa)
* **BAM_file**: a bam file of the Hi-C alignment (e.g. MAP_SORTED.bam)
* **COV_file**: a txt file of contigs' coverage information computed by script: ‘jgi summarize bam contig depths’ from MetaBAT2 (e.g. depth.txt)


#### Example
```
python ./hiczin.py norm -e HindIII -e NcoI -v final.contigs.fa MAP_SORTED.bam depth.txt out
```


### Output file
All outputs of HiCzin normalization pipeline are located in the OUT_directory you specified when running the software. 

* **hiczin.log**: the specific implementation information of HiCzin pipeline
* **contig_info.csv**: information of contigs, containing four columns (contigs' name, the number of restriction sites on contigs, contigs' length 
and coverage) or three columns (contigs' name, length and coverage)
* **Normalized_contact_matrix.npz**: a sparse matrix of normlized Hi-C contact maps in csr format and can be reloaded using **scipy.sparse.load_npz('Normalized_contact_matrix.npz')**
* **valid_contact.csv**: information of valid contacts, containing three columns (index of the first contig, index of the second contig, and value of raw valid  intra-species contacts)
* **HiCzin_normalized_contact.gz**: Compressed format of the normalized contacts and contig information by pickle. This file can further serve as the input of [**HiCBin**](https://github.com/dyxstat/HiCBin) and [**HiFine**](https://github.com/dyxstat/HiFine).


## Contacts and bug reports
If you have any questions or suggestions, welcome to contact Yuxuan Du (yuxuandu@usc.edu).


## Copyright and License Information
This program is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

This program is distributed in the hope that it will be useful, but WITHOUT ANY WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.

You should have received a copy of the GNU General Public License along with this program. If not, see http://www.gnu.org/licenses/.







