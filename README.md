# ngs-pipeline
NGS pipeline for calling somatic variants from whole exome sequencing (WES) data using GATK Best Practices and Mutect2

## Author
* Tomas Bencomo ([https://tjbencomo.github.io](https://tjbencomo.github.io))

## Prerequisites
Before you start setting up the pipeline, make sure you have your reference genome assembled.
The GRCh38 (hg38) genome is available on the Broad's
GATK [website](https://software.broadinstitute.org/gatk/download/bundle).

You'll also need the `cache` files for 
[Variant Annotation Predictor (VEP)](https://github.com/Ensembl/ensembl-vep).
Follow the tutorial 
[here](https://uswest.ensembl.org/info/docs/tools/vep/script/vep_cache.html#cache) 
to download the data files.
Don't forget to index the files before running the pipeline.
## Setup

1. Create a new Github repository using this workflow as a template with the `Use this template` button
at the top of this page. This will allow you to track any changes made to the analysis with `git`
2. Clone the repository to the machine where you want to perform data analysis
3. Edit `samples.csv` and `units.csv` with the details for your analysis.
See the `schemas/` directory for details about each file.
4. Configure `config.yml`. See `schemas/config.schema.yaml` for info about each required field. Note that each sample 
represents one patient. There should be normal and tumor sequencing data for each
sample. Each sample should have two rows in `units`, one normal row and one tumor row. Sequencing data must be
paired, so both `fq1` and `fq2` are required.

## Environments
`snakemake` is required to run `ngs-pipeline`, and other programs (`samtools`, `gatk`, etc)
are required for various steps in the pipeline. There are many ways to manage the required
executables.

### Singularity Container + Conda Environments
`snakemake` can run `ngs-pipeline` in a `singularity` container. Inside this container
each step is executed with a `conda` environment specified in `envs/`. This approach
controls the OS and individual packages, ensuring that certain software versions are
used for analysis. This approach can be enabled with the `--use-conda --use-singularity`
flags. **This approach is recommended because using YAMLs to specify environment records the
software version used for each step, helping others reproduce your results.**

### Other
Although `conda` and `singularity` are recommended, as long as all the packages are installed
on your machine, the pipeline will run. You can also only use `conda` environments and
skip the `singularity` container with `--use-conda`, although this can create difficulties
reproducing results.

## Usage
After finishing the setup and enabling the `conda` environment, inside the analysis directory with
`Snakefile` do a dry run to check for errors
```
snakemake -n
```
Once you're ready to run the analysis navigate to the base directory with `Snakefile` and type
```
snakemake --use-conda --use-singularity
```
If your machine has multiple cores, you can use these cores with
```
snakemake -j [cores]
```
This will run multiple rules simultaneously, speeding up the analysis.

The pipeline produces two key files: `mafs/variants.maf` and `qc/multiqc_report.html`.
`variants.maf` includes somatic variants from all samples that passed Mutect2 filtering.
They have been annotated with VEP and labeled by [VCF2MAF](https://github.com/mskcc/vcf2maf). 
`multiqc_report.html` includes quality metrics like coverage for the fully processed BAM files. 
Individual VCF files for each sample prior to VCF2MAF mapping are named `{sample}.vcf` in `vcfs/`.


### Cluster Execution
If you're using a compute cluster, you can take advantage of massively
parallel computation to speed up the analysis. Only SLURM clusters are
currently supported, but if you work with another cluster system (SGE etc)
`snakemake` makes it relatively easy to add cluster support.

Follow these instructions to enable SLURM usage
1. Edit the `out` field in `cluster.json` to tell SLURM where to save pipeline `stdout` and `stderr`.
2. Run `snakemake` from the command line with the following options
```
snakemake --cluster-config cluster.json -j 100 --cluster 'sbatch -p {cluster.partition} -t {cluster.time} --mem {cluster.mem}  -c {cluster.ncpus} -o {cluster.out}'
```
This command can be run in an `sbatch` job.
In the `sbatch` script, don't forget to `conda activate`
before running snakemake.

If for some reason you can't leave the master `snakemake` process running, `snakemake`
offers the ability to launch all jobs using `--immediate-submit`. This
approach will submit all jobs to the queue immediately and finish the master `snakemake`
process. The downside of this method is that temporary files are not supported, so
the results directories will be very cluttered. 
To use `--immediate-submit` follow these steps
1. Give `parseJobID.sh` permission to run
```
chmod +x parseJobID.sh
```
2. Submit the `snakemake` jobs
```
snakemake --cluster-config cluster.json --cluster 'sbatch $(./parseJobID.sh {dependencies}) -t {cluster.time} --mem {cluster.mem} -p {cluster.partition} -c {cluster.ncpus} - o {cluster.out}' --jobs 100 --notemp --immediate-submit
```

## Test Dataset
A small sample of `chr21` reads are supplied from the 
[Texas Cancer Research Biobank](http://txcrb.org/index.html) for
end-to-end pipeline tests. 
This data is made available as open access data with minimal privacy
restrictions. Please read the [Conditions of Use](http://txcrb.org/data.html)
before using the data.

## Citations
This pipeline is based on `dna-seq-gatk-variant-calling` by 
[Johannes Köster](https://github.com/snakemake-workflows/dna-seq-gatk-variant-calling).
If you use this pipeline, make sure to cite references for all of the tools used in the workflow:
```
snakemake
gatk
samtools
mosdepth
fastqc
multiqc
vep
vcf2maf
```
Citations to be added...
