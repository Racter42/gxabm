# Automated Benchmarking in Galaxy
An opinionated Python Bioblend script for automating benchmarking tasks in Galaxy.

## Installation

1. Clone this repository.
   ```bash
   git clone https://github.com/galaxyproject/gxabm.git
   cd gxabm
   ```
1. Create a virtual env and install the required libraries
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install --upgrade pip
   pip install -r requirements.txt
   ```


### Credentials

You will need an [API key](https://training.galaxyproject.org/training-material/faqs/galaxy/preferences_admin_api_key.html) for every Galaxy instance you would like to intereact with. You will also need the *kubeconfig* file for each Kubernetes cluster.  The `abm` script loads the Galaxy server URLs, API keys, and the location of the *kubeconfig* files from a Yaml configuration file that it expects to find in `$HOME/.abm/profile.yml` or `.abm-profile.yml` in the current directory.  You can use the `profile-sample.yml` file as a starting point and it includes the URLs for all Galaxy instances we have used to date (December 22, 2021 as of this writing). 

> :exclamation: **NOTE** by default `kubectl` expects that all *kubeconfig*s to be stored in a single configuration file located at `$HOME/.kube/config`. However, this is a global, system wide, configuration file meaning two shells can not operate on different Kubernetes clusters at the same time.  Therefore the `abm` scripts expects each cluster to store it's configuration in its own *kubeconfig* file.  I store mine in a directory named `$HOME/.kube/configs`, although this is just a personal preference and they can be stored anywhere.

## Usage

To get general usage information run the command:

```bash
> python3 abm.py help
```

You can get information about a specific `abm` command with:

```bash
> python3 abm.py workflow help
```

> The included `setup.sh` file can be *sourced* to both activate the virtual environment and create an alias so you do not need to type `python3 abm.py` all the time.  The remainder of this document assumes that the `setup.sh` file has been *sourced*

```bash
> source setup.sh
> abm workflow help
```

When running a command (i.e. not just printing help) you will need to specify the Galaxy instance to target as the first parameter:

```bash
> abm aws workflow list
> abm aws workflow run configs/paired-dna.yml
```

## New In 2.0.0

Version 2.0.0 refactors the `workflow` and `benchmark` commands to eliminate any confusion between a Galaxy *workflow* and what `abm` referred to as a *workflow*.  

### Terms and Definitions

**workflow**<br/>
A [Galaxy workflow](https://galaxyproject.org/learn/advanced-workflow/). Workflows in `abm` are mangaged with the `workflow` sub-command. Workflows can **not** be run directly via the `abm` command, but are run through the *benchmark* or *experiment* commands.

**benchmark**<br/>
A *benchmark* consists of one or more *workflows* with their inputs and outputs defined in a YAML configuration file. See the [Benchmark Configuration](#benchmark-configuration) section for instructions on defining a *benchmark*.

**experiment**<br/>
An *experiment* consists of one or more benchmarks to be run on one or more cloud providers. Each experiment definition consists of:
1. The number of *runs* to be executed. Each *benchmark* will be executed this number of times.
1. The *benchmarks* to be executed
1. The cloud providers the *benchmarks* should be executed on
1. The job rule configurations to be used. The job rule configurations define the number of CPUs and amount of memory to be allocated to the tools being benchmarked.

See the [Experiment Configuration](#experiment-configuration) section for instructions on defining an *experiment*.

### Changes to Functionality
While the functionality in `abm` is the same, some functions have been moved to other sub-commands.  In particular, the `workflow translate`, `workflow validate`, and `workflow run` command have been moved to the `benchmark` subcommand and the `benchmark run` and `benchmark summarize` commands have moved to the `experiment` subcommand.

| 1.x | 2.x |
|-----|-----|
| workflow translate | benchmark translate |
| workflow validate | benchmark validate |
| workflow run | benchmark run |
| benchmark run | experiment run |
| benchmark summarize | experiment summarize |


## Benchmark Configuration

The runtime parameters for benchmarking runs are specified in a YAML configuration file.  The configuration file can contain more than one runtime configuration specified as a YAML list. This file can be stored anywhere, but several examples are included in the `config` directory. 

The YAML configuration for a single workflow looks like:

```yaml
- workflow_id: d6d3c2119c4849e4
  output_history_base_name: RNA-seq
  reference_data:
    - name: Reference Transcript (FASTA)
      dataset_id: 50a269b7a99356aa
  runs:
    - history_name: 1
      inputs:
      - name: FASTQ RNA Dataset
        dataset_id: 28fa757e56346a34
    - history_name: 2
      inputs:
      - name: FASTQ RNA Dataset
        dataset_id: 1faa2d3b2ed5c436
```

- **workflow_id**<br/> 
  The ID of the workflow to run.
  
- **output_history_ base_name**  (optional)<br/>
  Name to use as the basis for histories created.  If the *output_history_base_name* is not specified then the  *workflow_id* is used.
  
- **reference_data** (optional)<br/>
  Input data that is the same for all benchmarking runs and only needs to be set once.  See the section on *inputs* below for a description of the fields

- **runs**<br/>
  Input definitions for a benchmarking run.  Each run defintion shoud contain:

  - **history_name** (optional) <br/>
    The name of the history created for the output.  The final output history name is generated by concatenating the *output_history_base_name* from above and the *history_name*.  If the *history_name* is not specified an incrementing integer counter is used.
  - **inputs**<br/>
    The one or more input datasets to the workflow.  Each input specification consists of:
    1. **name** the input name as specified in the workflow editor
    2. **dataset_id** the History API ID as displayed in the workflow editor or with the  `abm history list` command.

## Experiment Configuration

Each *experiment* is defined by a YAML configuration file. Example experiments can be found in the `experiments` directory. 

```yaml
name: Benchmarking DNA
runs: 3
benchmark_confs:
  - benchmarks/dna-named.yml
cloud:
  - tacc1
  - tacc2
job_configs:
  - 4x8
  - 8x16
```
- **name**<br/>
The name of the experiment.  This value is not currently not used.
- **runs**<br/>
The number of times each *benchmark* will be executed.  **Note** a *benchmark* configuration may itself define more than one *workflow* execution.
- **benchmark_confs**<br/>
The *benchmark* configurations to be execute during the *experiment*. These directories/files are expected to be relative to the current working directory.
- **cloud**<br/>
The cloud providers, as defined in the `profile.yml` file, where the experiments will be run.  The cloud provider instances must already have the *workflows* and history datasets uploaded and available for use.  Use the `bootstrap.py` script to provision an instance for running experiements.
- **job_configs**<br/>
The `jobs.rules.container_mapper_rules` files that define the CPU and memory resources allocated to tools.  These files must be located in the `rules` directory.

## Moving Workflows

Use the `abm <cloud> workflow download` and `abm <cloud> workflow upload` commands to transfer Galaxy *workflows* between Galaxy instances.

```bash
> abm cloud1 workflow download <workflow ID> /path/to/save/workflow.ga
> abm cloud2 workflow upload /path/to/save/workflow.ga
```

**NOTE** the name of the saved file (workflow.ga in the above example) is unrelated to the name of the workflow as it will appear in the Galaxy user interface or when listed with the `workflow list` command.

## Moving Benchmarks

The `benchmark translate` and `benchmark validate` commands can be used when moving workflows and datasets between Galaxy instances.  The `benchmark translate` command takes the path to a *benchmark* configuration file, translates the workflow and dataset ID values to their name as they appear in the Galaxy user interface, and writes the configuration to stdout.  To save the translated workflow configuration, redirect the output to a file:

```bash
> abm aws benchmark translate config/rna-seq.yml > benchmarks/rna-seq-named.yml
```

Then use the `benchmark validate` command to ensure that the other Galaxy instance has the same workflow and datasets installed:

```bash
> abm gcp benchmark validate config/rna-seq-named.yml
```

## Moving Histories

### Exporting Histories

1. Ensure the history is publicly available (i.e. published) on the Galaxy instance.  You can do this through the Galaxy user interface or via the `abm history publish` command:
```bash
$> abm cloud history publish <history id>
```
If you do not know the `<history id>` you can find it with `abm cloud history list`.

2. Export the history
```bash
$> abm cloud history export <history id>
```
Make note of the URL that is returned from the `histroy export` command as this is the URL to use to import the history to another Galaxy instance. Depending on the size of the datasets in the history it may take several hours for the history to be exported, during which time your computer terminal will be blocked.  Use the `[-n|--no-wait]` option if you do not want `history export` to block until the export is complete.
```bash
$> abm cloud history export <history id> --no-wait
```
The `history export` command will return immediately and print the job ID for the export job.  Use this job id to obtain the status of the job and determine when it has completed.
```bash
$> abm cloud job show <job id>
```
Once a history has been exported the first time, and as long it has not changed, running `abm history export` again simply print the URL and exit without re-exporting the history.  This is useful when the `--no-wait` option was specified and we need to determine the URL to use for importing.

> :heart: A History should only be exported once and the URL re-used on new benchmarking instances as they are created. Use the `lib/histories.yml` file to record the URLs so they can be easily reused with the `history import` command.

### Importing Histories
To import a history use the URL returned from the `history export` command.
```bash
$> abm dest history import URL

# For example
$> abm dest history import https://usegalaxy.org/history/export_archive?id=9198b7907edea3fa&jeha_id=02700395dbc14520
```
We can easily import histories defined in `lib/histories.yml` by specifying the YAML dictionary key name. 
```bash
$> abm dest history import rna
```

## Troubleshooting

Generate SSL/TLS certificates used by `kubeadm`.  Use the `--apiserver-cert-extra-sans` parameter to list additional IP addresses that the certificates will be valid for.

```bash
> kubeadm init phase certs all --apiserver-advertise-address=0.0.0.0 --apiserver-cert-extra-sans=10.161.233.80,114.215.201.87
````

## Future Work

- Run benchmarks/experiments in parallel when using more than one cloud provider.
- Integrate with the [Galaxy Benchmarker](https://github.com/usegalaxy-eu/GalaxyBenchmarker)
- Use as much as we can from [Git-Gat](https://github.com/hexylena/git-gat)

## Contributing

Fork this repository and then create a working branch for yourself from the `dev` branch. All pull requests should target `dev` and not the `master` branch.

```bash
git clone https://github.com/ksuderman/bioblend-scripts.git
cd bioblend-scripts
git checkout -b my-branch
```

If you decide to work on one of the [issues](gxabm/issues) be sure to assign yourself to that issue to let others know the issue is taken.

