# MBL STAMPS Assembly - a tutorial

C. Titus Brown, STAMPS 2017

Note, this is [an excerpt from a several-day workshop](http://2017-dibsi-metagenomics.readthedocs.io/) on shotgun metagenomics.

Learning goals:

* show the process (and input/output requirements) for running an assembly;
* do some minimal characterization of an assembly
* quantify the contigs output by the assembler in each of two samples using salmon

## Data set

We'll be using a subset of the data from the mesothermic reservoir samples (SB1 and SB2) from the [Hu et al., 2016](http://mbio.asm.org/content/7/1/e01669-15.full) study - "Genome-Resolved Metagenomic Analysis Reveals Roles for Candidate Phyla and Other Microbial Community Members in Biogeochemical Transformations in Oil Reservoirs." In addition to being a relatively low-complexity community, there are paired samples that we can use to do differential abundance analysis of contigs.

## Let's run an assembler!

Log into your class server etc. etc.

First, load the appropriate environment.
```
. /class/stamps-shared/sourmash/load.sh
# this lets use use khmer and sourmash as well
# as doing a 'module load molevol'

cd ~/
mkdir assembly
cd assembly

ln -fs /class/stamps-shared/titus-assembly/SRR*.fq.gz .

/class/stamps-shared/megahit/megahit --memory 10e9 --num-cpu-threads 8 \
 --12 SRR1976948.abundtrim.subset.pe.fq.gz,SRR1977249.abundtrim.subset.pe.fq.gz \
    -o combined
```

This will take 20-40 minutes.  If we are out of time, we can also cancel out with CTRL-C and do: `cp /class/stamps-shared/titus-assembly/combined/final.contigs.fa combined/`.

You can take a look at the output of the assembler with:
```
head combined/final.contigs.fa
```

Things we can discuss while assembling:

* what format are the input files in?
* [how well does assembly work?](https://osf.io/4a2k3/) -- presentation
* where should I do my computation?
* what is the output of an assembler?


## Extract sequences over 500 bp

Next, let's take only those contigs longer than 500 bp, as shorter contigs are not useful for gene identification.  This uses the `extract-long-sequences.py` from [the khmer software package](https://khmer.readthedocs.io); see [the command line docs](https://khmer.readthedocs.io/en/stable/user/scripts.html) for more info.

Extract long contigs:

```
extract-long-sequences.py -l 500 combined/final.contigs.fa > long-contigs.fa
```

## Get some basic statistics using Quast:

Here we use [QUAST](http://bioinf.spbau.ru/quast) to get some basic statistics on the assembly:

```
/class/stamps-shared/titus-assembly/quast/quast.py long-contigs.fa

cat quast_results/latest/report.txt
```

## Quantify the contigs using Salmon

Next, we are going to quantify the abundance of each of the output contigs in the two original samples, using [Salmon](https://salmon.readthedocs.io/en/latest/). Salmon is normally a *transcriptome* quantification tool but here we are using it for metagenome quantification - an "off-label" approach endorsed by the authors.

First, build an index for the assembly:
```
/class/stamps-shared/salmon/bin/salmon index \
    -t long-contigs.fa -i combined_index
```

Next, split the read data sets into "R1" and "R2" reads - this uses `split-paired-reads.py` from [the khmer software package](https://khmer.readthedocs.io).

```
for i in *.fq.gz; do split-paired-reads.py $i; done

# rename the files
mv SRR1976948.abundtrim.subset.pe.fq.gz.1 SRR1976948.1.fq
mv SRR1976948.abundtrim.subset.pe.fq.gz.2 SRR1976948.2.fq

mv SRR1977249.abundtrim.subset.pe.fq.gz.1 SRR1977249.1.fq
mv SRR1977249.abundtrim.subset.pe.fq.gz.2 SRR1977249.2.fq
```

Now, for each sample, quantify the abundance of the assembled contigs *in that sample*:

```
/class/stamps-shared/salmon/bin/salmon quant -i combined_index \
    --libType IU -1 SRR1976948.1.fq -2 SRR1976948.2.fq \
    -o SRR1976948.quant

/class/stamps-shared/salmon/bin/salmon quant -i combined_index \
    --libType IU -1 SRR1977249.1.fq \
    -2 SRR1977249.2.fq -o SRR1977249.quant
```

This now gives us two output directories, `*.quant`, each one containing information about the abundance of the contigs in that sample.

Collect the counts output:

```
curl -L -O https://raw.githubusercontent.com/ngs-docs/2016-metagenomics-sio/master/gather-counts.py
python2 ./gather-counts.py
```

and convert it into a file that contains all the relevant info in one file, suitable for plotting:

```
for file in *counts
do
   name=${file%%.*}
   echo "processing $name"
   sed -e "s/count/$name/g" $file > tmp
   mv tmp $file
done
paste *counts |cut -f 1,2,4 > combined-counts.tsv
```

This file can now be downloaded, loaded into Excel and plotted.

You can also [view my copy on github](https://github.com/ngs-docs/2017-mbl-stamps/blob/master/combined-counts.tsv), or [download it here](https://raw.githubusercontent.com/ngs-docs/2017-mbl-stamps/master/combined-counts.tsv).

----

Note - above, we quantified the *contigs*.  If you want gene-specific abundance quantification, you can do so after annotation; see [these tutorials](https://2017-dibsi-metagenomics.readthedocs.io/en/latest/#thursday-day-4) that involve first annotation with Prokka.

Questions at end of period, if time --

* how would you validate/evaluate the abundance quantification?
* what would next steps be for making use of the abundance quant?
