### Getting (re)started

Double check if the directories (~/data/graph_genomes) exist (persists):

    ls -lh ~/Project_8
    
And all the software we installed is still available:

    vg
    bcftools
    jq -h
    dot -h
    Bandage -h 

If your environement is not persistent (resets everytime you log out), rerun the following commands, otherwise skip directly to the exercise (Working with larger graphs):

#### Installing the required packages

First you will need to aquire the necessary packages using the conda package manager:

    conda install -c conda-forge mamba
    mamba install -c conda-forge -c bioconda vg jq graphviz bcftools Bandage

vg is a large collection of tools that is under very active development. If you want to learn more about vg visit the [vg homepage](https://github.com/vgteam/vg). 

To test if things are working smoothly, you can run:

    vg

#### Getting the required data

In this exercise, let's step up to a slightly bigger example .Make sure you have checked out vg (again if the environement isn't persistent):

    mkdir ~/Project_8
    cd ~/Project_8
    git clone https://github.com/vgteam/vg.git

Now create a directory to work on for this tutorial:

    mkdir exercise2
    cp vg/test/1mb1kgp/* exercise2/
    cd exercise2

### Working with larger graphs


This directory contains 1Mbp of 1000 Genomes data for chr20:1000000-2000000. As for the tiny example, let's' build one linear graph that only contains the reference sequence and one graph that additionally encodes the known sequence variation. The reference sequence is contained in `1mb1kgp/z.fa`, and the variation is contained in `1mb1kgp/z.vcf.gz`. Make a reference-only graph named `ref.vg`, and a graph with variation named `z.vg`. Look at the previous examples to figure out the command.

You might be tempted to visualize these graphs (and of course you are welcome to try), but they are sufficiently big already that neato can run out of memory and crash.

In a nutshell, mapping reads to a graph is done in two stages: first, seed hits are identified and then a sequence-to-graph alignment is performed for each individual read. Seed finding hence allows vg to spot candidate regions in the graph to which a given read can map potentially map to. To this end, we need an index. In fact, vg needs two different representations of a graph for read mapping XG (a succinct representation of the graph) and GCSA (a k-mer based index). To create these representations, we use `vg index` as follows.

    vg construct -r z.fa -v z.vcf.gz -m 32 >z.vg
    vg index -x z.xg z.vg
    vg index -g z.gcsa -k 16 z.vg

Passing option `-k 16` tells vg to use a k-mer size of *16k*. The best choice of *k* will depend on your graph and will lead to different trade-offs of sensitivity and runtime during read mapping.

As mentioned above, the whole graph is unwieldy to visualize. But thanks to the XG representation, we can now quickly **find** individual pieces of the graph. Let's extract the vicinity of the node with ID 2401 and create a PDF.

    vg find -n 2401 -x z.xg -c 10 | vg view -dp - | dot -Tpdf -o 2401c10.pdf

    
2401c10.pdf![image](https://user-images.githubusercontent.com/1767457/116373399-74790080-a84c-11eb-8f77-95d57aa2beb2.png)


The option `-c 10` tells `vg find` to include a context of 10 nodes in either direction around node 2401. You are welcome to experiment with different parameter to `vg find` to pull out pieces of the graph you are interested in.  

Next, we want to play with mapping reads to the graph. Luckily, vg comes with subcommand to simulate reads off off the graph. That's done like this:

    vg sim -x z.xg -l 100 -n 1000 -e 0.01 -i 0.005 -a > z.sim

This generates 1000 (`-n`) reads of length (`-l`) with a substitution error rate of 1% (`-e`) and an indel error rate of 0.5% (`-i`). Adding `-a` instructs `vg sim` to output the true alignment paths in GAM format rather than just the plain sequences. Map can work on raw sequences (`-s` for a single sequence or `-r` for a text file with each sequence on a new line), FASTQ (`-f`), or FASTA (`-f` for two-line format and `-F` for a reference sequence where each sequence is over multiple lines).

We are now ready to map the simulated read to the graph.

    vg map -x z.xg -g z.gcsa -G z.sim >z.gam

We can visualize alignments using an option to `vg view`. The format is not pretty but it provides us enough information to understand the whole alignment.
More advanced visualization methods (like [IVG](https://vgteam.github.io/sequenceTubeMap/)) are in development, but do not work on the command line.

These commands would show us the first alignment in the set:

    vg view -a z.gam | head -1 | vg view -JaG - > first_aln.gam
    vg find -x z.xg -G first_aln.gam | vg view -dA first_aln.gam - | dot -Tpdf -o first_aln.pdf

first_aln.pdf![image](https://user-images.githubusercontent.com/1767457/116373438-7cd13b80-a84c-11eb-9b57-e55adf916198.png)


We see the `Mappings` of the `Alignment` written in blue for exact matches and yellow for mismatches above the nodes that they refer to. Many alignments can be visualized at the same time. A simpler mode of visualization `vg view -dSA` gives us the alignment's mappings to nodes, colored in the range from green to red depending on the quality of the match to the particular node.

To visualize many alignments, we can sort and index our GAM and then pull out the alignments matching a particular subgraph.

    vg gamsort z.gam | vg gamsort -i z.sort.gam.gai - >z.sort.gam
    vg find -n 1020 -x z.xg -c 10 >z.sub.vg
    vg find -l z.sort.gam -A z.sub.vg >z.sub.gam
    vg view -dA z.sub.gam z.sub.vg | dot -Tpdf -o z1020gam.pdf

z1020gam.pdf![image](https://user-images.githubusercontent.com/1767457/116373469-83f84980-a84c-11eb-98d0-36530e1499ce.png)

For evaluation purposes, vg has the capability to compare the newly created read alignments to true paths of each reads used during simulation.

    vg map -x z.xg -g z.gcsa -G z.sim --compare -j

This outputs the comparison between mapped and and true locations in JSON format. We can use this quickly check if our alignment process is doing what we expect on the variation graph we're working on. For instance, we could set alignment parameters that cause problems for our alignment and then observe this using the `--compare` feature of the mapper. For example, we can map two ways and see a difference in how correct our alignment is:

    vg map -x z.xg -g z.gcsa -G z.sim --compare -j | jq .correct | sed s/null/0/ | awk '{i+=$1; n+=1} END {print i/n}'

In contrast, if we were to set a very high minimum match length we would throw away a lot of the information we need to make good mappings, resulting in a low correctness metric:

    vg map -k 51 -x z.xg -g z.gcsa -G z.sim --compare -j | jq .correct | sed s/null/0/ | awk '{i+=$1; n+=1} END {print i/n}'

It is essential to understand that our alignment process works against the graph which we have constructed. This pattern allows us to quickly understand if the particular graph and configuration of the mapper produce sensible results at least given a simulated alignment set. Note that the alignment comparison will break down if we simulate from different graphs, as it depends on the coordinate system of the given graph.

### Exploring the benefits of graphs for read mapping

To get a first impression of how a graph reference helps us do a better job while mapping reads. We will construct a series of graphs from a linear reference to a graph with a lot variation and look at mapping rates, i.e. at the fraction of reads that can successfully be mapped to the graph. For examples, we might include variation above given allele frequency (AF) cutoffs and vary this cutoff. You can make a VCF with a minimum allele fequency with this command (replace `FREQ` with the frequency you want):
    
    # get required tools for next steps
    conda install -c bioconda samtools bwa bcftools sambamba 
    bcftools filter -i 'AF > FREQ' z.vcf.gz > min_af_filtered.vcf

The ``--exclude`` option in conjunction with custom [expressions](https://samtools.github.io/bcftools/bcftools-man.html#expressions) is particularly useful to this end. You may also want to think about other properties that would be useful to filter on.

With this input you should be able to run the whole pipeline:

- Construct the graphs with the filtered VCF
- Index the graphs for mapping
- Map the simulated reads to the graphs
- Check the identity of the read mappings (use `jq .identity` rather than `jq .correct`)

Try doing this on graphs with a range of minimum allele frequencies (e.g. 0.5, 0.1, 0.01, etc.). How do the properties of the mappings change with different minimum frequencies? Let's also look at the sizes of the graphs and indexes.

    ls -sh *.vg
    ls -sh *.gcsa*

How do these files seem to scale with the minimum cutoff? Note that the identity metric is flawed, in that we are not asking if the mapping itself is more accurate, just if it matches the graph better in the aggregate.

### BONUS: Mapping data from real data to examine the improvement
DUE TO A RESTRUCTURE OF THE NIST-GAIB RESTRUCTURE OF THE FTP SERVER, YOU ARE UNABLE TO STREAM THE BAM FILE LINKED BELLOW. BUT THE INSTRUCTIONS HOLD TRUE STILL.

We can also download some real data mapping to this region to see if the different graphs provide varying levels of performance.

    samtools view -b ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftpNA12878/NIST_NA12878_HG001_HiSeq_300x/RMNISTHS_30xdownsample.bam 20:1000000-2000000 >NA12878.20_1M-2M.30x.bam
    # alternatively:
    # note: if you are taking this course, this data is available in /home/ubuntu/workshop/2_Tue_Garrison/human
    wget http://hypervolu.me/~erik/tmp/HG002-NA24385-20_1M-2M-50x.bam

The first is a sample that was used in the preparation of the 1000 Genomes results, and so we expect to find it in the graph. The second wasn't used in the preparation of the variant set, but we do expect to find almost all of its variants in the 1000G set. Why is this true?

We can run a single-ended alignment test to compare with bwa mem:

    samtools fastq -1 HG002-NA24385-20_1M-2M-50x_1.fq.gz -2 HG002-NA24385-20_1M-2M-50x_2.fq.gz HG002-NA24385-20_1M-2M-50x.bam
    bwa index z.fa
    bwa mem z.fa HG002-NA24385-20_1M-2M-50x_1.fq.gz | sambamba view -S -f json /dev/stdin | jq -cr '[.qname, .tags.AS] | @tsv' > bwa_mem.scores.tsv
    bcftools filter -i 'AF > 0.01' z.vcf.gz >z.AF0.01.vcf
    vg construct -r z.fa -v z.AF0.01.vcf -m 32 >z.AF0.01.vg
    vg index -x z.AF0.01.xg -g z.AF0.01.gcsa z.AF0.01.vg
    vg map --drop-full-l-bonus -d z.AF0.01 -f HG002-NA24385-20_1M-2M-50x_1.fq.gz -j | pv -l | jq -cr '[.name, .score] | @tsv' >vg_map.AF0.01.scores.tsv

Then we can compare the results using sort and join:

    join <(sort bwa_mem.scores.tsv ) <(sort vg_map.AF0.01.scores.tsv ) | awk '{ print $0, $3-$2 }' | tr ' ' '\t' | sort -n -k 4 | pv -l | gzip >compared.tsv.gz

We can then see how many alignments have improved or worse scores:

    zcat compared.tsv.gz | awk '{ if ($4 < 0) print $1 }' | wc -l
    zcat compared.tsv.gz | awk '{ if ($4 == 0) print $1 }' | wc -l
    zcat compared.tsv.gz | awk '{ if ($4 > 0) print $1 }' | wc -l

In general, the scores improve. Try plotting a histogram of the differences to see the extent of the effect.

We can pick a subset of reads with high or low score differentiation to realign and compare:

    zcat HG002-NA24385-20_1M-2M-50x_1.fq.gz | awk '{ printf("%s",$0); n++; if(n%4==0) { printf("\n");} else { printf("\t\t");} }' | grep -Ff <(zcat compared.tsv.gz | awk '{ if ($4 < -10) print $1 }' ) | sed 's/\t\t/\n/g' | gzip >worse.fq.gz
    zcat HG002-NA24385-20_1M-2M-50x_1.fq.gz | awk '{ printf("%s",$0); n++; if(n%4==0) { printf("\n");} else { printf("\t\t");} }' | grep -Ff <(zcat compared.tsv.gz | awk '{ if ($4 > 10) print $1 }' ) | sed 's/\t\t/\n/g' | gzip >better.fq.gz

Let's dig into some of the more-highly differentiated reads to understand why vg is providing a better (or worse) alignment. How might you go about this? There are many ways you could do this, but you may find some of these commands useful:

- `vg view -aj ALN.gam` : convert a .gam alignment into a text-based JSON representation with one alignment per line
- `vg view -aJG ALN.json` : convert the JSON representation back into a .gam alignment file
- `vg mod -g ID -x N GRAPH.vg` : extract the subgraph that is within `N` nodes from node `ID`
- `vg mod -P -i ALN.gam GRAPH.vg` : add the paths from the alignment into the graph (similar to the reference path in the exercise)

