# Shotgun metagenomics workshop 2018

## Obtaining a cloud computing instance

For this demo, we'll be using the Google Cloud Platform.  The procedure would be
very similar on Amazon's service, or on another cloud computing provider.
Google provides new users with a substantial credit towards computing costs,
and allows us to launch a terminal window conveniently from the web browser.

We will request a VM instance with 8 CPU cores and 52 GB of memory.  We'll ask
for 100 GB of hard drive space.  The operating system will be Ubuntu Linux
18.04 LTS.  This is a typical Linux system, with no special bioinformatics
software installed. (Make sure you don't select the "Minimal" version.)

We will install all the software we need in your home directory using a system
called Conda. This approach requires no administrative priveleges, so you
should be able to use it almost anywhere. After installation, we'll obtain our
data, configure the pipeline, and run the pipeline to generate output files and
a report.

## Installation

These steps are copy/pasted from the Sunbeam documentation at
http://sunbeam.readthedocs.io/en/latest/quickstart.html

To open a terminal window in your browser, select the "Open in browser window" 
option under the "Connect" column on your Google Cloud VM instances page. 
Execute the following commands:

```{bash}
cd ~
git clone -b stable https://github.com/sunbeam-labs/sunbeam sunbeam-stable
ls
```

```{bash}
cd sunbeam-stable
bash install.sh
```

Follow instructions, and execute the command shown.

```{bash}
echo "export PATH=$PATH:/home/kylebittinger/miniconda3/bin" >> ~/.bashrc
```

Following the instructions, close and re-open the terminal window.

```{bash}
source activate sunbeam
```

## Upload data files

Our data files are deposited at NCBI's Sequence Read Archive (SRA). If you
Google "conda sra", you will find that your first hit is for a software package
named "sra-tools".  Let's execute the installation command from that page to
install this software.

```{bash}
conda install -c bioconda sra-tools
```

Using `sra-tools`, we can download samples by accession number.  We will grab a
small number of samples from the PLEASE study (Pediatric Longitudinal Study of
Semi-Elemental Diet and Stool Microbiome), which was conducted at Penn and CHOP.

```{bash}
cd ~
mkdir workshop-data
cd workshop-data
fasterq-dump SRR2145310
fasterq-dump SRR2145329
fasterq-dump SRR2145381
fasterq-dump SRR2145353
fasterq-dump SRR2145354
fasterq-dump SRR2145492
fasterq-dump SRR2145498
```

First mini-lecture.

## Initialize the project

```{bash}
cd ~
mkdir workshop-project
sunbeam init workshop-project --data_fp workshop-data
```

Look at the project directory.

```{bash}
cd workshop-project
ls
```

Explore the samples file.

```{bash}
nano samples.csv
```

Explore the configuration file.

```{bash}
nano sunbeam_config.yml
```


## Download reference data

We need two reference databases to run this sample: a database of host DNA
sequence to remove, and a database of bacterial DNA to match against.

We'll get the human genome data from UCSC.  Filtering against the entire human
genome takes too long, so we'll only filter against chromosome 1. The following
commands download and unzip the .fasta file for chromosome 1:

```{bash}
cd ~
mkdir human
cd human
wget http://hgdownload.cse.ucsc.edu/goldenPath/hg38/chromosomes/chr1.fa.gz
gunzip chr1.fa.gz
```

Sunbeam requires that the host DNA sequence files end in ".fasta", so it can
find them automatically.

```{bash}
mv chr1.fa chr1.fasta
```

The database of bacterial genomes comes pre-built from the homepage of our
taxonomic assignment software, Kraken.

```{bash}
cd ~
wget https://ccb.jhu.edu/software/kraken/dl/minikraken_20171101_4GB_dustmasked.tgz
tar xvzf minikraken_20171101_4GB_dustmasked.tgz
```

Now that we have reference databases, we need to add them to our configuration
file.

```{bash}
cd ~
nano workshop-project/sunbeam_config.yml
```

The configuration values are below.  You'll need to navigate to the right spot
in your configuration file, and substitute the directory name "kylebittinger"
with your google username.

```
  host_fp: "/home/kylebittinger/human"
```

```
  kraken_db_fp: "/home/kylebittinger/minikraken_20171101_4GB_dustmasked"
```

## Run the pipeline

We are ready to actually run the pipeline.  All the information about how
to run the pipeline is in our configuration file, so we'll provide that to
Sunbeam (`--configfile` argument).  We'll also let Sunbeam know how many CPU
cores we'd like to use (`--jobs` argument).

```{bash}
cd ~
sunbeam run --configfile workshop-project/sunbeam_config.yml --jobs 7
```

## Generate a report

```{bash}
sudo apt update
sudo apt install r-base
```

To look at some of our results, we'll install a Sunbeam extension and generate
a report.

```{bash}
cd ~/sunbeam-stable/extensions
git clone https://github.com/sunbeam-labs/sbx_report
conda install --file sbx_report/requirements.txt
```

This is going to install the R programming language on our remote computer,
which will take a bit of time.  When the installation is complete, 

```{bash}
cd ~
sunbeam run --configfile workshop-project/sunbeam_config.yml final_report
```
