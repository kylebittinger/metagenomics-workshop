# Run the analysis pipeline

> Note: run these commands on the *remote computer*, not on your
> laptop.

> If you're re-connecting to your Google Cloud instance, please run
> `source activate sunbeam` to "turn on" your analysis software.

We have installed the software, uploaded our data set, downloaded
reference databases, and configured the pipeline to run.  Now, we are
ready to actually run the pipeline and produce a set of output files.

To run the pipeline, we need to be in the *software* directory, not
our project directory.

```bash
cd ~/sunbeam-master
ls
```

Listing the files, we should see the configuration file
`deadmice-config.yml`, produced in the last section of the tutorial.
We need to tell our pipeline to use this configuration file, and tell
it how many CPU cores to use in parallel.

The command to run the analysis pipeline is:

```bash
snakemake --configfile deadmice-config.yml --cores 8
```

Upon success, you should see loads of text scrolling up your terminal
screen.  Some text is printed from the different commands, some
consists of messages from the pipeline software.  We\'ll go over this
in the workshop session.

When I tested this exaple data set, it took about 1 hour and 15
minutes to run, from start to finish.
