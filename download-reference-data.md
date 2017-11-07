# Downloading the reference data

> Note: run these commands on the *remote computer*, not on your
> laptop.

```bash
cd ~
ls
```

```bash
cd deadmice
ls
```

```bash
mkdir indexes
cd indexes
```

The first thing we need is the mouse reference genome.  We\'re going
to grab this from NCBI\'s RefSeq collection, which contains a curated
set of high-quality reference genomes.  Starting at the the main
RefSeq site at https://www.ncbi.nlm.nih.gov/refseq/, I clicked the
link for "RefSeq genomes FTP" to go to the download area.  Then, I
clicked on the folder for `vertebrate_mammalian/`, then scrolled down
for the folder `Mus_musculus/`, then `reference/` for the main
reference genome for that organism.  In here, there\'s only one link,
which doesn\'t look like a directory, but it\'s a directory.  Clicking
here takes us to a set of files for the latest mouse reference genome.
We want the genome sequence in FASTA format,
`GCF_000001635.26_GRCm38.p6_genomic.fna.gz`.

We don\'t have a web browser installed on our Google Cloud computer,
so we\'ll copy the link and use a program called `wget` to download
the file in our remote computer.  We\'ll unzip the file and rename it
as `mouse.fasta` while we\'re at it.

```bash
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/vertebrate_mammalian/Mus_musculus/reference/GCF_000001635.26_GRCm38.p6/GCF_000001635.26_GRCm38.p6_genomic.fna.gz
gunzip GCF_000001635.26_GRCm38.p6_genomic.fna.gz
mv GCF_000001635.26_GRCm38.p6_genomic.fna mouse.fasta
```

Filtering against the whole genome takes too long, so we\'re going to
remove reads hitting Chromosome 1 only.  I\'ve pre-filtered the data
files to remove reads matching to other parts of the mouse genome.

```bash
perl -ne 'print if 1../^>NT_166280\.1/' mouse.fasta|grep -v "NNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNNN"|head -n -1 > mouse_chr1.fasta
```

Now that we have the sequence for mouse chromosome 1, we\'ll allow out
alignment software to "index" this file for efficient searching.

```bash
bwa index mouse_chr1.fasta
```

Each sequencing run includes a bit of phage phi X 174; we\'ll download
this genome as well, to remove any cross-contamination.  The steps are
similar to the mouse genome.

```bash
wget ftp://ftp.ncbi.nlm.nih.gov/genomes/refseq/viral/Enterobacteria_phage_phiX174_sensu_lato/latest_assembly_versions/GCF_000819615.1_ViralProj14015/GCF_000819615.1_ViralProj14015_genomic.fna.gz
gunzip GCF_000819615.1_ViralProj14015_genomic.fna.gz
mv GCF_000819615.1_ViralProj14015_genomic.fna phix174.fasta
bwa index phix174.fasta
```

Next, we\'ll download a reference database for our taxonomic
assignment software.  In the special topics session, I\'ll say more
about how this software works.

The `.tgz` format is another kind of compressed file, like ZIP files.
Thus, we\'ll use a different program to "unzip" the data here.

```bash
wget https://ccb.jhu.edu/software/kraken/dl/minikraken_20171101_8GB_dustmasked.tgz
tar xvzf minikraken_20171101_8GB_dustmasked.tgz
mv minikraken_20171101_8GB_dustmasked mindb
```

There are a number of specialized gene databases that can be used with
shotgun metagenomic data.  Here, we\'ll download CARD, a database of
antibiotic resistance genes (https://card.mcmaster.ca/).

We\'re going through the same routine, this time with different
alignment software and yet another compressed file format.

```bash
mkdir card
cd card
wget https://card.mcmaster.ca/download/0/broadstreet-v1.2.1.tar.bz2
tar xvjf broadstreet-v1.2.1.tar.bz2
makeblastdb -dbtype nucl -in nucleotide_fasta_protein_homolog_model.fasta
cd ..
```

Our last step, before we run the pipeline, is to write a configuration
file for our analysis.  This involves another bit of command line
magic, because I don\'t know the path to your home directory without
asking the computer.

Most of the code here is the actual contents of the configuration
file. You don\'t need to understand the format, but notice how we
point to our reference data directories and specify parameters for
each step.

```bash
cat << EOF > ${WORKDIR}/sunbeam-master/deadmice-config.yml
# Sunbeam configuration file
#
# Paths:
#   Paths are resolved through the following rules:
#     1. If the path is absolute, the path is parsed as-is
#     2. If the path is not absolute, the path at 'root' is appended to it
#     3. If the path is not 'output_fp', the path is checked to ensure it exists
#
# Suffixes:
#   Each subsection contains a 'suffix' key that defines the folder under
#   'output_fp' where the results of that section are put.
#

# General options
all:
  root: "${WORKDIR}/deadmice/"
  data_fp: "data_files"
  output_fp: "sunbeam_output"
  filename_fmt: "{sample}_{rp}.fastq"
  subcores: 4
  exclude: []


# Quality control
qc:
  suffix: qc
  # Trimmomatic
  threads: 4
  java_heapsize: 512M
  leading: 3
  trailing: 3
  slidingwindow: [4,15]
  minlen: 36
  adapter_fp: "${WORKDIR}/miniconda3/envs/sunbeam/share/trimmomatic/adapters/NexteraPE-PE.fa"
  # Cutadapt
  fwd_adapters: ['GTTTCCCAGTCACGATC', 'GTTTCCCAGTCACGATCNNNNNNNNNGTTTCCCAGTCACGATC']
  rev_adapters: ['GTTTCCCAGTCACGATC', 'GTTTCCCAGTCACGATCNNNNNNNNNGTTTCCCAGTCACGATC']
  # Decontam.py
  pct_id: 0.5
  frac: 0.6
  keep_sam: False
  method: bwa
  human_genome_fp: "indexes/mouse_chr1.fasta"
  phix_genome_fp: "indexes/phix174.fasta"


# Taxonomic classifications
classify:
  suffix: classify
  threads: 4
  kraken_db_fp: "indexes/mindb"
  taxa_db_fp: ""


# Contig assembly
assembly:
  suffix: assembly
  min_length: 300
  threads: 4
  cap3_fp: "${WORKDIR}/sunbeam-master/local/CAP3"


# Contig annotation
annotation:
  suffix: annotation
  min_contig_len: 500
  circular_kmin: 10
  circular_kmax: 1000
  circular_min_len: 3500


# Blast databases
blast:
  threads: 4

blastdbs:
  root_fp: "indexes/"
  nucleotide:
    card:
      card/nucleotide_fasta_protein_homolog_model.fasta


mapping:
  suffix: mapping
  genomes_fp: ""
  igv_fp: "${WORKDIR}/sunbeam-master/local/IGV/igv"
  threads: 4
  keep_unaligned: False
  igv_prefs:
    # Smooth rendered text.  By default it looks jagged.
    ENABLE_ANTIALIASING: true
    # The pixel width of the left panel that shows the name of the input files.
    # The default is a bit too narrow and will truncate long filenames.
    NAME_PANEL_WIDTH: 360
    # The alignment window size, in kb, below which alignments will become
    # visible.  We don't want to ever require zooming so we will set it to a
    # high value.
    SAM.MAX_VISIBLE_RANGE: 1000
EOF
```

