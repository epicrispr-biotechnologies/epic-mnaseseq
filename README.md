# ![nf-core/mnaseseq](docs/images/nf-core-mnaseseq_logo.png)

[![GitHub Actions CI Status](https://github.com/nf-core/mnaseseq/workflows/nf-core%20CI/badge.svg)](https://github.com/nf-core/mnaseseq/actions)
[![GitHub Actions Linting Status](https://github.com/nf-core/mnaseseq/workflows/nf-core%20linting/badge.svg)](https://github.com/nf-core/mnaseseq/actions)
[![Nextflow](https://img.shields.io/badge/nextflow-%E2%89%A519.10.0-brightgreen.svg)](https://www.nextflow.io/)

[![install with bioconda](https://img.shields.io/badge/install%20with-bioconda-brightgreen.svg)](http://bioconda.github.io/)
[![Docker](https://img.shields.io/docker/automated/nfcore/mnaseseq.svg)](https://hub.docker.com/r/nfcore/mnaseseq)
[![Cite with Zenodo](http://img.shields.io/badge/DOI-10.5281/zenodo.6581372-1073c8)](https://doi.org/10.5281/zenodo.6581372)

## Introduction

This is an unmodified version of **nfcore/mnaseseq**, a bioinformatics analysis pipeline used for DNA sequencing data obtained via micrococcal nuclease digestion. It has a modified `config` file and `awsbatch` profile for Epicrispr.

The pipeline is built using [Nextflow](https://www.nextflow.io), a workflow tool to run tasks across multiple compute infrastructures in a very portable manner. It comes with docker containers making installation trivial and results highly reproducible.

## Pipeline summary

1. Raw read QC ([`FastQC`](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/))
2. Adapter trimming ([`Trim Galore!`](https://www.bioinformatics.babraham.ac.uk/projects/trim_galore/))
3. Alignment ([`BWA`](https://sourceforge.net/projects/bio-bwa/files/))
4. Mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
5. Merge alignments from multiple libraries of the same sample ([`picard`](https://broadinstitute.github.io/picard/))
    1. Re-mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
    2. Filtering to remove:
        * reads mapping to blacklisted regions ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/), [`BEDTools`](https://github.com/arq5x/bedtools2/))
        * reads that are marked as duplicates ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that arent marked as primary alignments ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that are unmapped ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads that map to multiple locations ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
        * reads containing > 4 mismatches ([`BAMTools`](https://github.com/pezmaster31/bamtools))
        * reads that are soft-clipped ([`BAMTools`](https://github.com/pezmaster31/bamtools))
        * reads that have an insert size within specified range ([`BAMTools`](https://github.com/pezmaster31/bamtools); *paired-end only*)
        * reads that map to different chromosomes ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
        * reads that arent in FR orientation ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
        * reads where only one read of the pair fails the above criteria ([`Pysam`](http://pysam.readthedocs.io/en/latest/installation.html); *paired-end only*)
    3. Alignment-level QC and estimation of library complexity ([`picard`](https://broadinstitute.github.io/picard/), [`Preseq`](http://smithlabresearch.org/software/preseq/))
    4. Create normalised bigWig files scaled to 1 million mapped reads ([`BEDTools`](https://github.com/arq5x/bedtools2/), [`bedGraphToBigWig`](http://hgdownload.soe.ucsc.edu/admin/exe/))
    5. Calculate genome-wide coverage assessment ([`deepTools`](https://deeptools.readthedocs.io/en/develop/content/tools/plotFingerprint.html))
    6. Call nucleosome positions and generate smoothed, normalised coverage bigWig files that can be used to generate occupancy profile plots between samples across features of interest ([`DANPOS2`](https://sites.google.com/site/danposdoc/))
    7. Generate gene-body meta-profile from DANPOS2 smoothed bigWig files ([`deepTools`](https://deeptools.readthedocs.io/en/develop/content/tools/plotProfile.html))
6. Merge filtered alignments across replicates ([`picard`](https://broadinstitute.github.io/picard/))
    1. Re-mark duplicates ([`picard`](https://broadinstitute.github.io/picard/))
    2. Remove duplicate reads ([`SAMtools`](https://sourceforge.net/projects/samtools/files/samtools/))
    3. Create normalised bigWig files scaled to 1 million mapped reads ([`BEDTools`](https://github.com/arq5x/bedtools2/), [`wigToBigWig`](http://hgdownload.soe.ucsc.edu/admin/exe/))
    4. Call nucleosome positions and generate smoothed, normalised coverage bigWig files that can be used to generate occupancy profile plots between samples across features of interest ([`DANPOS2`](https://sites.google.com/site/danposdoc/))
    5. Generate gene-body meta-profile from DANPOS2 smoothed bigWig files ([`deepTools`](https://deeptools.readthedocs.io/en/develop/content/tools/plotProfile.html))
7. Create IGV session file containing bigWig tracks for data visualisation ([`IGV`](https://software.broadinstitute.org/software/igv/)).
8. Present QC for raw read and alignment results ([`MultiQC`](http://multiqc.info/))

## EC2
This pipeline uses DSL1 and requires `nextflow` <=22.10, which cannot use AWS SSO credentials with IMDSv2. If you authenticate with AWS through SSO, you will have to allow IMDSv1 on your EC2 instance to run this pipeline. This has reduced security compared to IMDSv2.

When creating a new EC2 instance: go to **Metadata version** and select “V1 and V2 (token optional)”

When editing a current EC2 instance: select instance, then Actions>Instance Settings>Modify instance metadata options and select “Optional” under **IMDSv2**.

## Quick start

i. Install [`Docker`](https://docs.docker.com/engine/installation/) or [`Singularity`](https://www.sylabs.io/guides/3.0/user-guide/). To use `Docker` on an Ubuntu EC2 instance, after installing the [`apt` package](https://docs.docker.com/engine/install/ubuntu/), you will have to [add your user to the `docker` group](https://docs.docker.com/engine/install/linux-postinstall) in order to run the pipeline.

ii. Create a conda environment with the correct version of `nextflow`.
``` bash
conda create --name mnase nextflow=22.10
conda activate mnase
```

iii. Download the pipeline and test it on a minimal dataset with a single command

```bash
cd epic-mnaseseq
nextflow run main.nf -profile test,docker
```

iv. Start running your own analysis. Use the `awsbatch` profile, as well as the `docker` profile to save to S3. Note that `nextflow=22.10` requires a local, non-S3, log directory.

```bash
nextflow run main.nf -profile awsbatch,docker \
--genome GRCh38 \
--input s3://[path]/input_design.csv \
-work-dir s3://[path]/Intermediate_files/ \
--outdir s3://[path]/Output_files/ \
--tracedir ./log/ 
```

See [usage docs](docs/usage.md) for all of the available options when running the pipeline.

## Documentation

The nf-core/mnaseseq pipeline comes with documentation about the pipeline, found in the `docs/` directory:

1. [Installation](https://nf-co.re/usage/installation)
2. Pipeline configuration
    * [Local installation](https://nf-co.re/usage/local_installation)
    * [Adding your own system config](https://nf-co.re/usage/adding_own_config)
    * [Reference genomes](https://nf-co.re/usage/reference_genomes)
3. [Running the pipeline](docs/usage.md)
4. [Output and how to interpret the results](docs/output.md)
5. [Troubleshooting](https://nf-co.re/usage/troubleshooting)

## Credits

The pipeline was originally written by [The Bioinformatics & Biostatistics Group](https://www.crick.ac.uk/research/science-technology-platforms/bioinformatics-and-biostatistics/) for use at [The Francis Crick Institute](https://www.crick.ac.uk/), London.

The pipeline was developed by [Harshil Patel](mailto:harshil.patel@crick.ac.uk).

Many thanks to others who have helped out along the way too, including (but not limited to): [@crickbabs](https://github.com/crickbabs).

## Contributions and Support

If you would like to contribute to this pipeline, please see the [contributing guidelines](.github/CONTRIBUTING.md).

For further information or help, don't hesitate to get in touch on [Slack](https://nfcore.slack.com/channels/mnaseseq) (you can join with [this invite](https://nf-co.re/join/slack)).

## Citation

If you use nf-core/mnaseseq for your analysis, please cite it using the following doi: [10.5281/zenodo.6581372](https://doi.org/10.5281/zenodo.6581372).

You can cite the `nf-core` publication as follows:

> **The nf-core framework for community-curated bioinformatics pipelines.**
>
> Philip Ewels, Alexander Peltzer, Sven Fillinger, Harshil Patel, Johannes Alneberg, Andreas Wilm, Maxime Ulysse Garcia, Paolo Di Tommaso & Sven Nahnsen.
>
> _Nat Biotechnol._ 2020 Feb 13. doi: [10.1038/s41587-020-0439-x](https://dx.doi.org/10.1038/s41587-020-0439-x).  
> ReadCube: [Full Access Link](https://rdcu.be/b1GjZ)

An extensive list of references for the tools used by the pipeline can be found in the [`CITATIONS.md`](CITATIONS.md) file.
