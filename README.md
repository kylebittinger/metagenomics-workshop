# Shotgun Metagenomics Workshop

VM: 50GB hard drive space, 12GB memory, must turn on networking, default linux image.

## 1. Download metagenomic data

First we need a data set to work on. We'll download a set of 10 fecal
and saliva samples that we've deposited on Zenodo, a site for
depositing scientific data.

Download and uncompress the data.

```bash
wget 'https://zenodo.org/records/10157719/files/saliva_feces_data.tar.gz?download=1' -O saliva_feces_data.tar.gz
tar xvzf saliva_feces_data.tar.gz
```

Check out the results.

```bash
ls -lh saliva_feces_data
zless saliva_feces_data/Feces.15_R1.fastq.gz
# Hit q to exit
```

Clean up.

```bash
rm saliva_feces_data.tar.gz
```

## 2. Download host genome

We're only going to download one chromosome, because it takes too long
to index the entire human genome for filtering during the
workshop. The data files have been pre-filtered to remove all the
other chromosomes.

```bash
wget 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi?db=nuccore&id=CM000663.2&rettype=fasta' -O humanch1.fasta
mkdir host
mv humanch1.fasta host
```

Check out the results.

```bash
less host/humanch1.fasta
# Hit q to exit
```

## 3. Install Conda

Conda is our system for installing third-party software needed for the
pipeline. Conda allows us to install whatever we need inside our home
directory, and places the software we need into separate environments
so the different software components don't clash with each other.

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-py311_23.10.0-1-Linux-x86_64.sh
bash Miniconda3-py311_23.10.0-1-Linux-x86_64.sh
# License: hit enter to continue, hit the spacebar until you get to the bottom
#   then type "yes" to accept terms
# Miniconda3 will be installed into this location: hit enter to confirm the location
# Do you wish to update your shell profile to automatically initialize conda?
#   type "yes" and hit return
```

At this point you'll be instructed to close and re-open the terminal
window. Please do this. When the window re-opens, you should see
"(base)" on the left-hand side of your prompt. This indicates that
you're in the base environment for Conda, and that your installation
was most likely successful.

Check out the results by listing your available environments. Right
now, the base environment is all you've got. Don't worry, we'll make
more soon.

```bash
conda env list
```

Clean up.

```bash
rm Miniconda3-py311_23.10.0-1-Linux-x86_64.sh
```

## 4. Install Sunbeam

Now we will install Sunbeam, the pipeline that we'll use for
bioinformatics processing.

```bash
wget https://github.com/sunbeam-labs/sunbeam/releases/download/v4.1.0/sunbeam.tar.gz
mkdir sunbeam4.1.0
tar xvzf sunbeam.tar.gz -C sunbeam4.1.0
cd sunbeam4.1.0
bash install.sh
cd
conda activate sunbeam4.1.0
```

The Conda environment for Sunbeam should now be activated. If so, you
will see "(sunbeam4.1.0)" in your command prompt.

```bash
sunbeam --help
```

Clean up.

```bash
rm sunbeam.tar.gz
```

## 5. Install Sunbeam Extension sbx_kraken

To make taxonomic assignments, we need to extend Sunbeam with one
piece of software, Kraken.

```bash
sunbeam extend https://github.com/sunbeam-labs/sbx_kraken
```

Let's check the extensions directory and make sure the files are
there.

```bash
ls sunbeam4.1.0/extensions
```

# 6. Download taxonomic database

```bash
wget https://genome-idx.s3.amazonaws.com/kraken/k2_standard_08gb_20231009.tar.gz
mkdir krakendb
tar xvzf k2_standard_08gb_20231009.tar.gz -C krakendb
```

```bash
rm k2_standard_08gb_20231009.tar.gz
```

# 7. Initialize and configure our project

```bash
sunbeam init workshop --data_fp saliva_feces_data
nano workshop/sunbeam_config.yml
# Change host_fp to "/home/yourusername/host"
# Change kraken_db_fp to "/home/yourusername/krakendb"
# Hit Control-x (^x) to exit, "y" to save modified buffer, and enter to confirm the filename
```

# 8. Run the pipeline to generate decontaminated FASTQ files

```bash
sunbeam run --profile workshop all_decontam
```

```bash
less workshop/sunbeam_output/qc/reports/preprocess_summary.tsv
# Hit q to exit
```

# 9. Run the pipeline to generate a taxonomic summary

```bash
sunbeam run --profile workshop all_classify
```

```bash
less workshop/sunbeam_output/classify/kraken/all_samples.tsv
# Hit q to exit
```

# 10. Spruce up the taxonomic summary

Prepare and activate a new Conda environment

```bash
# prompt should say "(sunbeam4.1.0)"
conda deactivate
# prompt should now say "(base)"
conda env create -f sunbeam4.1.0/extensions/sbx_kraken/sbx_kraken_env.yml
conda activate sbx_kraken
# prompt should now say "(sbx_kraken)"
```

Filter the table to remove any remaining human reads. You will see a couple of warning messages from the last command but you can safely ignore them.

```bash
biom table-ids --observations -i workshop/sunbeam_output/classify/kraken/all_samples.biom > saliva_feces_observation_ids.txt
grep -vx 9606 saliva_feces_observation_ids.txt > saliva_feces_observation_ids_nohuman.txt
biom subset-table -a observation -s saliva_feces_observation_ids_nohuman.txt -i workshop/sunbeam_output/classify/kraken/all_samples.biom -o saliva_feces_nohuman.biom
```

Add metadata to the taxonomic summary file in BIOM format.


```bash
wget https://raw.githubusercontent.com/kylebittinger/metagenomics-workshop/updates-2024/saliva_feces_metadata.txt
biom add-metadata -m saliva_feces_metadata.txt --output-as-json -i saliva_feces_nohuman.biom -o saliva_feces_finished.biom
```

```bash
less saliva_feces_finished.biom
# Hit G to see the bottom of the file
# Hit q to quit
```

