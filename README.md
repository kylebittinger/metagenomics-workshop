# Shotgun Metagenomics Workshop 2024

To carry out our bioinformatics work, we'll order up a virtual machine
(VM) from Google Cloud. Once you're signed up, go to the console. In
the "Quick Access" area, click on the box for "Compute Engine." The
screen should now say "Compute Engine" in the upper left. Next to
"Compute Engine," it should say "VM instances." This is your dashboad
for managing virtual machines.

We need to create a new virtual machine instance to do our work, so
click on "CREATE INSTANCE." We need to change a few settings for our
bioinformatics work. Here's a summary of the important stuff:

* Change the name to something you like.
* The default machine configuration, E2, is fine.
* Under "Machine type," go to the "CUSTOM" tab and crank up the
  memory all the way to 16 GB.
* The default number of cores is fine for this workshop.
* Under "Boot disk," click on "CHANGE" and set the size to 50 GB.
* The default operating system and version, Debian GNU/Linux 11
  (bullseye), is fine for this workshop.
* Under "Firewall," check the boxes for "Allow HTTP traffic" and
  "Allow HTTPS traffic."

The machine will take a minute to start up. Once it does, you'll see a
button that says "SSH" under the "Connect" column on the right. When
you click on this button, your web browser will launch a command-line
terminal window. We are now ready to get started with bioinformatics.

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
```

```bash
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
workshop. We'll use a trick to remove the remaining human reads at the
end of the workflow.

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
# License agreement: hit return to see the license, then hit the spacebar
#   until you get to the bottom, then type "yes" to accept the terms.
# Miniconda3 will be installed into this location: hit return to confirm the
#   location.
# Do you wish to update your shell profile to automatically initialize conda?
#   Type "yes" and hit return.
```

At this point you'll be instructed to close and re-open the terminal
window. Please do this.

When the window re-opens, you should see "(base)" on the left-hand
side of your prompt. This indicates that you're in the base
environment for Conda, and that your installation was most likely
successful.

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
```

We need to change directories temporarily to install Sunbeam. This is
the only time we'll change directories during the workshop. After you
install Sunbeam, it's very important that you run the `cd` command to
go back to your home directory.

```bash
cd sunbeam4.1.0
bash install.sh
cd
```

Let's activate the Conda environment for Sunbeam.

```bash
conda activate sunbeam4.1.0
```

The Conda environment for Sunbeam should now be activated. If so, you
will see "(sunbeam4.1.0)" in your command prompt. Let's ask Sunbeam
what it can do.

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

## 6. Download taxonomic database

We need to download a reference database for the taxonomic
assignments.

```bash
wget https://genome-idx.s3.amazonaws.com/kraken/k2_standard_08gb_20231009.tar.gz
mkdir krakendb
tar xvzf k2_standard_08gb_20231009.tar.gz -C krakendb
```

Let's look inside the `krakendb` directory to see what files are
there.

```bash
ls krakendb
```

Clean up.

```bash
rm k2_standard_08gb_20231009.tar.gz
```

## 7. Initialize and configure our project

```bash
sunbeam init workshop --data_fp saliva_feces_data
nano workshop/sunbeam_config.yml
# Change host_fp to "/home/yourusername/host"
# Change kraken_db_fp to "/home/yourusername/krakendb"
# Hit Control-x (^x) to exit, then "y" to save modified buffer,
#   then hit return to confirm the filename
```

## 8. Run the pipeline to generate decontaminated FASTQ files

```bash
sunbeam run --profile workshop all_decontam
```

```bash
less workshop/sunbeam_output/qc/reports/preprocess_summary.tsv
# Hit q to exit
```

## 9. Run the pipeline to generate a taxonomic summary

```bash
sunbeam run --profile workshop all_classify
```

```bash
less workshop/sunbeam_output/classify/kraken/all_samples.tsv
# Hit q to exit
```

## 10. Spruce up the taxonomic summary

We need to make some revisions to the file of taxonomic assignments
before we're ready to move on with the downstream analysis. For this
work, we'll create a new Conda environment from the template that
comes with our `sbx_kraken` extension.

```bash
# prompt should say "(sunbeam4.1.0)"
conda deactivate
# prompt should now say "(base)"
conda env create -f sunbeam4.1.0/extensions/sbx_kraken/sbx_kraken_env.yml
conda activate sbx_kraken
# prompt should now say "(sbx_kraken)"
```

The first thing we'll do is filter the table to remove any remaining
human reads. When we ran the pipeline, remember that we only removed
part of chromosome 1. As you run the last command, you will see a
couple of warning messages, but you can safely ignore them.

```bash
biom table-ids --observations -i workshop/sunbeam_output/classify/kraken/all_samples.biom > saliva_feces_observation_ids.txt
grep -vx 9606 saliva_feces_observation_ids.txt > saliva_feces_observation_ids_nohuman.txt
biom subset-table -a observation -s saliva_feces_observation_ids_nohuman.txt -i workshop/sunbeam_output/classify/kraken/all_samples.biom -o temp_nohuman.biom
```

The second thing we'll do is to add some sample metadata to the BIOM
file.

```bash
wget https://raw.githubusercontent.com/kylebittinger/metagenomics-workshop/updates-2024/saliva_feces_metadata.txt
biom add-metadata -m saliva_feces_metadata.txt --output-as-json -i temp_nohuman.biom -o temp_metadata.biom
```

Finally, we need to fix up the species-level assignments. By default,
the species-level taxonomic assignments only include the specific
name, and are missing the generic name. This makes them hard to
decipher when we do the downstream bioinformatic analysis. Usually, we
would take care of this later in the analysis, but for today, we'll
run a command to fix the names right in the BIOM file. We're doing
some "command line magic" here, so don't worry too much about
deciphering the command.

```bash
perl -072 -pe's/g__([^"]+)", "s__([^"]+)/g__$1", "s__$1 $2/' temp_metadata.biom > saliva_feces.biom
```

We have our finished product. Let's see what the file looks like.

```bash
less saliva_feces.biom
# Hit G to see the bottom of the file
# Hit q to quit
```

Clean up.

```bash
rm temp_nohuman.biom
rm temp_metadata.biom
```

## 11. Download the finished output file

As a last step in the bioinformatics workflow, we're going to use the
handy download button to download the taxonomic assignments to our
laptops for use with MicrobiomeDB. You will need to enter the absolute
filepath in the popup window. Use this command to print out the
absolute filepath, then copy it to your clipboard before hitting the
Download button.

```bash
realpath saliva_feces.biom
```
