# IPA HiFi Genome Assembler

<h1 align="center"><img width="512px" src="doc/IPA-logo-3.png"/></h1>

## Description

This repo contains the implementation of the IPA HiFi Genome Assembler.
It's currently implemented as a Snakemake workflow (`workflow/ipa.snakefile`) and runs the following stages:
1. Building the SeqDB and SeedDB from the input reads.
2. Overlapping using the Pancake overlapper.
3. Phasing the overlaps using the Nighthawk phasing tool.
4. Filtering the overlaps using Falconc m4Filt.
5. Contig construction using Falcon's `ovlp_to_graph` and `graph_to_contig` tools.
6. Read tracking for read-to-contig assignment.
7. Polishing using Racon.

For more info: https://github.com/PacificBiosciences/pbbioconda/wiki/Improved-Phased-Assembler


## Installation from Bioconda
IPA is available via Bioconda:
```
conda create -n ipa
conda activate ipa
conda install pbipa
```
IPA requires python 3.7+. Please use miniconda3, not miniconda2. We currently only support Linux 64-bit, not MacOS or Linux 32-bit.

## Usage examples
IPA can be run using the `ipa` executable. The runner contains two subtools:
- `ipa local` - Running a local job on the current machine.
- `ipa dist` - Running the assembly on the cluster.

Simplest example to run a phased assembly with polishing on a local machine looks like this:
```
ipa local -i <myreads.fasta>
```

Specifying custom number of threads and number of parallel jobs can provide more optimal performance on your machine:
```
ipa local --nthreads 20 --njobs 4 -i <myreads.fasta>
```
Note: this will use 80 CPUs for parallel jobs like overlapping, phasing and polishing (given that the input dataset is large enough).

Example of a distributed run on a cluster using SGE:
```
mkdir -p <out_dir>/qsub_log
ipa dist -i <myreads.fastq> --run-dir <out_dir> --cluster-args "qsub -S /bin/bash -N ipa -cwd -q default -pe smp {params.num_threads} -e qsub_log/ -o qsub_log/ -V"
```
Note: `--cluster-args` are passed directly to Snakemake. For a custom queue, please edit that string. Also, for other types of cluster environment, please consult the Snakemake documentation.

More details can be found here: https://github.com/PacificBiosciences/pbbioconda/wiki/Improved-Phased-Assembler

## Advanced Usage

Only one file (config) is required to run IPA:
- (mandatory) Config file: `config.json`.
- (optional) Input FOFN in case the config points to it: `input.fofn`.

Once the config file is specified, the workflow can be run via Snakemake directly, similar to this:
```
snakemake -p -j 1 -d RUN -s <ipa.snakefile> --configfile config.json --config MAKEDIR=.. -- finish
```

Additionally, the `ipa` runner tool provides an option `--only-print` which will not run the workflow, but instead only prints the Snakemake run command to the user. It also generates the config file from the command line options.
For example:
```
$ ipa local -i input.fofn --only-print
...
python3 -m snakemake -j 4 -d RUN -p -s /mnt/software/i/ipa/develop/etc/ipa.snakefile --configfile RUN/config.yaml --reason
```

The user can then copy this line and run Snakemake manually.

The same option is available in the cluster runner subtool `ipa dist`:
```
$ ipa dist -i input.fofn --run-dir RUN --cluster-args 'qsub -S /bin/bash -N ipa -cwd -q sequel-farm -pe smp {params.num_threads} -e qsub_log/ -o qsub_log/ -V' --nthreads 24 --njobs 40 --nshards 40 --only-print
...
python3 -m snakemake -j 40 -d RUN -p -s /mnt/software/i/ipa/develop/etc/ipa.snakefile --configfile RUN/config.yaml --reason --cluster 'qsub -S /bin/bash -N ipa -cwd -q sequel-farm -pe smp {params.num_threads} -e qsub_log/ -o qsub_log/ -V' --latency-wait 60 --rerun-incomplete
```

### Config file
Config can be specified either in JSON or YAML formats (as supported by Snakemake).

The structure of `config.json` is given here:
```
{
    "reads_fn": "input.fofn",
    "genome_size": 0,
    "coverage": 0,
    "advanced_options": "",
    "polish_run": 1,
    "phase_run": 1,
    "nproc": 8,
    "max_nchunks": 40,
    "tmp_dir": "/tmp"
}
```

Explanation of each parameter:
- `reads_fn`: Can be a FOFN, FASTA, FASTQ, BAM or XML. Also, gzipped versions of FASTA and FASTQ are available.
- `genome_size`: Used for downsampling in combination with `coverage`. If `genome_size * coverage <=0` downsampling is turned off.
- `coverage`: Used for downsampling in combination with `genome_size`. If `genome_size * coverage <=0` downsampling is turned off.
- `advanced_options`: A single line listing advanced options in the form of `key = value` pairs, separated with `;`.
- `polish_run`: Polishing will be applied if the value of this parameter is equal to `1`.
- `phase_run`: Phasing will be applied if the value of this parameter is equal to `1`.
- `nproc`: Number of threads to use on each compute node.
- `max_nchunks`: Parallel tasks will be groupped into at most this many chunks. If there is more than one task per chunk, they are executed linearly. Each chunk is executed in parallel. Useful for throttling job submissions.
- `tmp_dir`: Temporary directory which will be used for some disk-based operations, like sorting.

An example config with a custom `ipa2.advanced_options` string:
```
{
    "reads_fn": "input.fofn",
    "genome_size": 0,
    "coverage": 0,
    "advanced_options": "config_seeddb_opt = -k 30 -w 80 --space 1; config_block_size = 2048; config_phasing_piles = 20000",
    "polish_run": 1,
    "phase_run": 1,
    "nproc": 8
}
```

The list of all options that can be modified via the `advanced_options` string and their defaults:
```
config_autocomp_max_cov=1
config_block_size=4096
config_coverage=0
config_existing_db_prefix=
config_genome_size=0
config_ovl_filter_opt=--max-diff 80 --max-cov 100 --min-cov 2 --bestn 10 --min-len 4000 --gapFilt --minDepth 4 --idt-stage2 98
config_ovl_min_idt=98
config_ovl_min_len=1000
config_ovl_opt=--one-hit-per-target --min-idt 96
config_phase_run=1
config_phasing_opt=
config_phasing_split_opt=--split-type noverlaps --limit 3000000
config_polish_run=1
config_seeddb_opt=-k 30 -w 80 --space 1
config_seqdb_opt=--compression 1
config_use_hpc=0
config_use_seq_ids=1
```
