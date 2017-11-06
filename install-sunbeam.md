# Installing the Sunbeam metagenomics pipeline

In this post, we'll install and test the Sunbeam metagenomics pipeline
on a Google Cloud instance.  Here, I assume that you've been able to
follow Dan's instructions in part 1 to launch a new instance and
connect via SSH.

## Launch the instance 

After launching, we need to install some system programs on the
computer.  For this, we'll use the `apt-get` utility provided with the
operating system.

```bash
sudo apt-get install unzip xvfb xdotool libxrender1 libxi6
```

## Download the Sunbeam software

We'll download the software from GitHub, where it is distributed.

```bash
wget https://github.com/eclarke/sunbeam/archive/master.zip
```

After the software is downloaded successfully, you'll have a new file
called `master.zip` in your home directory.  You can verify this with
the `ls` command. Now that the file is downloaded, we'll decompress
the file with `unzip`.

```bash
unzip master.zip
```

## Install the software

We move into the software directory, where the installation scripts
are located.

```bash
cd sunbeam-master
```

Now, we're ready to actually install the Sunbeam analysis pipeline.
Sunbeam provides a script for this purpose.

```bash
./install.sh
./install_igv.sh
```

The pipeline uses a system called Conda to manage all the
bioinformatics analysis programs. This system works by storing all the
programs inside your home directory, so that we can use special
versions of programs just for this pipeline. We'll tell the computer
where to find the Conda system.

```bash
echo "export PATH=$HOME/miniconda3/bin:\$PATH" >> ~/.bashrc
source ~/.bashrc
```

To actually use Sunbeam, we'll "turn on" the bioinformatics software.

```bash
source activate sunbeam
```

This is the command you'll want to remember for future sessions.  Each
time you log into your cloud instance, you'll need to activate the
pipeline with `source activate sunbeam`.  Upon activation, you should
see that your command prompt begins with "(sunbeam)".

## Run the tests

Now that our software is installed and activated, we'll run the tests
included with Sunbeam to make sure everything is OK.

```bash
bash tests/test.sh
```

As the tests are running, you should see messages scrolling by on your
screen. This is a preview of what you might see when the actual
pipeline is running, so we'll devote some time in our session to
understanding the messages.

Our next step is to download our data files and get moving with some
real data analysis!
