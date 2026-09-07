# Cerebro

Metagenomic diagnostics pipeline and collaborative reporting stack for pathogen detection, species identification, host genome analysis, quality assurance and deployment in clinical and public health production environments.

<details>
<summary>🩸 Metagenomic diagnostic core functions </summary>
<br>
 
- Multi-classifier taxonomic profiling, metagenome assembly and alignment in Nextflow
- Optimized pangenome host depletion and background depletion with [`Scrubby`]()
- Viral infections, pan-viral enrichment protocols and syndrome-specific subtyping panels using [`Vircov`]()
- Differential host tumor DNA diagnostics using segmental CNV detection (supplementary)
- Custom database and index construction, grafted taxonomies, genome cleaning with [`Cipher`]()

</details>

<details>
<summary>📰 Collaborative clinical reporting (Bug Board) </summary>
<br>
 
- [Collaborative and auditable pathogen determination]() from metagenome sequencing results
- Multi-tenant Svelte application and API with secure local or web-server deployment configs
- Scalable application stack deployment with different data security and collaboration models
- Stack configuration and deployment integrated into the primary command-line interface ([Cerebro CLI]()) 
- Clinical reporting with [`Typst`]() formatted templates linked into evidence from multi-classifier/databases
- Secure [`wasm` enabled report generation]() in-browser for sensitive reports, interactive data visualizations
- Training interface for sample collectiosn with orthogonal reference data

</details>

<details>
<summary>🏥 Clinical and public health production environments </summary>
<br>
 
- Simulations using in silico syndromic reference panels for signal- and read-level data with [`Cipher`]()
- Evaluation of simulation and patient datasets for quality assurance with [`Cipher`]() and [`Cerebro`]()
- Background/sample site/kitome contamination issues in clinical or public health environments 
- Distributed sequence and analysis storage, file system and data retention policies, cloud storage etc. through [`SeaweedFS`]()

</details>

## Table of contents

- [Getting started](#getting-started)
- [Nextflow pipeline](#nextflow-pipeline)
  - [Requirements](#requirements)
  - [Entry points](#entry-points)
  - [Profiles](#profiles)
  - [Quick start](#quick-start)
  - [Apptainer and SLURM](#apptainer-and-slurm)
  - [Non-production run (default)](#non-production-run-default)
  - [Production mode](#production-mode)
  - [Outputs](#outputs)
  - [Known issues and cleanup](#known-issues-and-cleanup)
- [Cipher database and protocol resources](#cipher-database-and-protocol-resources)
- [Cerebro CLI](#cerebro-cli)
- [Clinical reporting](#clinical-reporting)
- [Application stack](#application-stack)
- [Cerebro API](#cerebro-api)
- [Cerebro FS](#cerebro-fs)
- [Databases and taxonomy](#databases-and-taxonomy)
- [Disclosure: AI-assisted diagnostic interpretation](#disclosure-ai-assisted-diagnostic-interpretation)
  - [Evaluation and its limits](#evaluation-and-its-limits)
  - [Reproducibility caveat](#reproducibility-caveat)
  - [Conditions of use](#conditions-of-use)
  - [Research code](#research-code)
- [Status](#status)


## Getting started

Let's step through some common tasks and core functions of `Cerebro` and the application and reporting stack. This section provides some examples of how to get started quickly with `Cerebro`. For more details and how to deploy and operate the full application in production please see the [documentation](). 

Minimum requirements:

* Linux OS
* Nextflow v24 or v25
* Conda/Mamba/Docker

Computational resource requirements are variable and range from a standard laptop for the application stack to full nation-wide server infrastructure for pipelines and web-application (if you were so inclined). This is because the application stack for data and reporting can be deployed with various [infrastructure, data security and collaboration models]() in mind and depends on the number of laboratories, collaborators, sequencing throughput, data storage and many other considerations.

> [!NOTE]
You do not need the `Docker` stack for core metagenome diagnostic pipelines and report generation - you can run the [Nextflow pipelines]() separately and use the [`Cerebro CLI`](#command-line-client) for data manipulation, processing of pipeline outputs and clinical report generation.

## Nextflow pipeline

### Requirements

- Linux, `Nextflow` v24 or v25
- `Mamba`/`Conda` (local executor) **or** `Apptainer` (recommended for HPC/SLURM)
- The `Cipher v2` database in `--databaseDirectory` (see [Databases and taxonomy](#databases-and-taxonomy))

### Entry points

Select the workflow with `-entry`:

| Entry | Purpose |
| --- | --- |
| `pathogen` | Pathogen detection and taxonomic profiling from paired-end Illumina reads. The main diagnostic workflow. |
| `panviral` | Panviral enrichment (probe hybridisation capture, e.g. Twist or Agilent). |
| `quality` | Quality control module only. |
| `culture` | Culture identification (under development). |
| `production` | Consumes staged sample files placed in `--stageDirectory` by `cerebro-tower`. **Not for interactive use**-see [Production mode](#production-mode) below, which is a different thing. |

### Profiles

Profiles are comma-separated and combine. You normally need one from each of the first three groups:

| Group | Profiles |
| --- | --- |
| **Execution** | `mamba`, `conda`, `apptainer` |
| **Assay** | `cns` (deduplication + Illumina adapters, for CSF/ocular fluid), `panviral` |
| **Database** | `cipher`, `cipher_nohost`, `ictv`, `ictv_nohost` |
| **Resources** | `micro` (8 cpu/32 GB), `tiny` (16/64), `mini` (32/128), `small` (64/512), `medium` (128/1024), `large` (192/1536), `xl` (256/1912), `dgx` |
| **Executor** | `slurm` (with `--slurmPartition`, `--slurmAccount`, `--slurmQos`, `--slurmGpu`) |
| **Modifiers** | `keepHost` (skip host depletion), `deduplicate` |

> [!NOTE]
> `dgx` is not only a resource profile-it also sets per-tool thread counts and the `Apptainer`
> library, cache and bind paths. Later profiles override earlier ones, so profile order matters.

### Quick start

Pathogen detection from paired-end Illumina reads of sterile-site specimens (validated for
cerebrospinal and ocular fluid):

```bash
nextflow run vidrl/cerebro \
  -revision v1.0.0 \
  -entry pathogen \
  -profile mamba,cns,cipher,medium \
  -work-dir ./work \
  --fastqPaired 'fastq/*_{R1_001,R2_001}.fastq.gz' \
  --databaseDirectory ./db \
  --outputDirectory 20260302
```

This runs quality control, taxonomic profiling and metagenome assembly for low microbial biomass
sample types, where distinguishing a pathogen from contamination and incidental background organisms
is the main diagnostic challenge (`needle-in-a-haystack`).

> [!WARNING]
> This pipeline is **not** suitable for high microbial biomass sample types (respiratory,
> environmental), where an abundant and diverse background microbiome is the main challenge
> (`haystack-full-of-needles`).

Symlink the database rather than copying it, it is roughly 2 TB:

```bash
mkdir 20260302_RUN && cd 20260302_RUN
ln -s /path/to/resources/CIPHER/cerebro ./db
mkdir fastq && cp /path/to/run/fastq/* fastq/
```

> [!IMPORTANT]
> Sample identifiers must follow `{SAMPLE_ID}__{NUCLEIC_ACID}__{SAMPLE_TAG}_{TAIL}`-for example
> `DW-63-103__DNA__S_S1`. Control tags `NTC`, `ENV` and `POS` are required for automatic control
> co-selection in the application, and DNA/RNA tags drive library pairing. Identifiers must be
> anonymised, must not contain spaces, and must not encode the sample site or type.

### Apptainer and SLURM

Some tools are not available through standard container channels and must be built locally into the
`Apptainer` library directory configured by the `dgx` profile (`$HOME/src/apptainer`):

```bash
mkdir -p ~/src/apptainer && cd ~/src
git clone https://github.com/vidrl/cerebro
cp cerebro/lib/resources/apptainer/*.def apptainer/
cd apptainer && for f in *.def; do apptainer build $(basename $f .def)-dev.sif $f; done
```

Then run with containers, optionally on SLURM:

```bash
nextflow run vidrl/cerebro \
  -revision v1.0.0 \
  -entry pathogen \
  -profile dgx,apptainer,slurm,cns,cipher \
  -work-dir ./work \
  --fastqPaired 'fastq/*_{R1_001,R2_001}.fastq.gz' \
  --databaseDirectory ./db \
  --outputDirectory 20260302
```


### Non-production run (default)

By default (`--production false`) the pipeline writes all results to `--outputDirectory` and uploads
nothing. Use this for evaluation, development, reprocessing, and any run where you do not want
results in the database. You can upload the resulting models later with `cerebro-client` (see
[Cerebro CLI](#cerebro-cli)).

### Production mode

Adding `--production` to a normal `-entry pathogen` run makes the pipeline build Cerebro data models
and upload them to a running stack at the end of the workflow. The target project is created if it
does not exist.

> [!NOTE]
> `--production` on `-entry pathogen` is **not** the same as `-entry production`. The `production`
> entry point consumes sample files staged by `cerebro-tower` from `--stageDirectory` and is used for
> automated ingest; `--production` simply enables the upload step of an ordinary run.

The stack must be up and you must be authenticated first. The pipeline reads the token from the
environment variable named by `--apiTokenEnv` (default `CEREBRO_API_TOKEN`):

```bash
# Authenticate against the API; you will be prompted for the password
export CEREBRO_API_TOKEN=$(cerebro-client --url https://api.example.org login -e you@example.org)

nextflow run vidrl/cerebro \
  -revision v1.0.0 \
  -entry pathogen \
  -profile dgx,apptainer,cns,cipher,medium \
  -work-dir ./work \
  --fastqPaired 'fastq/*_{R1_001,R2_001}.fastq.gz' \
  --databaseDirectory ./db \
  --outputDirectory 20260302 \
  --production \
  --apiUrl 'https://api.example.org' \
  --teamName 'CNS' \
  --databaseName 'Production' \
  --projectName '260129_VH01133_85_AAHK5TTM5' \
  --runName '260129_VH01133_85_AAHK5TTM5'
```

Production parameters:

| Parameter | Required | Notes |
| --- | --- | --- |
| `--production` |-| Enables model creation and upload. Off by default. |
| `--apiUrl` | yes | API endpoint address. |
| `--apiTokenEnv` | no | Environment variable holding the token. Default `CEREBRO_API_TOKEN`. |
| `--teamName` | yes | Must already exist. |
| `--databaseName` | yes | Created if absent. |
| `--projectName` | yes | Created if absent. Use the sequencing run identifier. |
| `--runName` | yes | Sequencing run identifier recorded on each model. |

All six are validated at workflow start and the run fails immediately if any is missing, so a
misconfigured production run does not waste compute.

> [!IMPORTANT]
> Upload one sequencing run per project. Workflow selection is not yet enforced in the interface, and
> uploading a duplicate sample identifier into a project will raise an error unless it came from a
> different workflow.


### Outputs

```
{outputDirectory}/
├── config.json              # pipeline configuration record for this run
├── quality/{sample_id}/     # raw quality-control outputs; QC'd FASTQ symlinked here
├── pathogen/{sample_id}/    # raw pathogen-detection outputs; assembled contigs, classifier reports
├── results/
│   ├── samples/             # per-sample summaries: {sample_id}.qc.json, {sample_id}.pd.json
│   ├── models/              # database models: {sample_id}.model.json (production runs)
│   ├── qc_reads.tsv         # read quality control summary, all samples
│   ├── qc_controls.tsv      # internal control alignments (synthetic and phage spike-ins)
│   ├── qc_background.tsv    # background depletion summary (not applicable by default)
│   └── species.tsv          # species-level profiling summary with lineage
└── models/{sample_id}.json  # aggregated models, production runs
```

`results/` holds the processed summaries that are safe to transfer to a remote server. The `quality/`
and `pathogen/` directories hold the full raw outputs and stay local.

The aggregation step that produces `results/models/` runs automatically under `--production`. To
reproduce it after a non-production run:

```bash
cerebro-client create-pathogen \
  --quality  results/samples/${sample_id}.qc.json \
  --pathogen results/samples/${sample_id}.pd.json \
  --taxonomy ${db_directory}/taxonomy \
  --pipeline-config config.json \
  --run-id   ${run_id} \
  -o ${sample_id}.model.json
```


### Known issues and cleanup

Other known issues:

- The `concoct` binning process may fail. This is expected and can be ignored; its output is not used
  in current revisions.
- `--skipAssembly` accepts a list of samples to exclude from metagenome assembly.

Once a run has completed and you no longer need `-resume`:

```bash
rm -r ./work                 # Nextflow working directory
rm -r ./fastq                # only if you copied rather than symlinked the reads
```

## Cipher database and protocol resources

The `Cipher v2` database is **not yet publicly distributed**. It is approximately 2 TB and contains the classifier indices, reference genome collections (GTDB, EuPath, WormBase, ICTV, CHM13v2) and the grafted taxonomy used by the pathogen detection pipeline. Contact the maintainers for access, or build it yourself (**experimental**) with [`Cipher`](https://github.com/vidrl/cipher).

Once you have the database, pass its directory to `--databaseDirectory`. Do **not** copy it into your execution directory, symlink it instead:

```bash
mkdir 20260302_RUN && cd 20260302_RUN
ln -s /path/to/resources/CIPHER/cerebro ./db
```

In addition to the database, the default short-read configuration requires:

- **human reference index** for alignment-based host depletion (`Scrubby`) usually CHM13v2
- **background/control sequences** for depletion in the quality control module for synthetic spike-ins (ERCC/EDCC) and internal phage controls

Public distribution of these resources is pending by the current maintainers. Until then, the pipeline can be run with host depletion and control alignment disabled, but quality control metrics and the tiered detection thresholds described in the preprint will not be comparable.

## Cerebro CLI

### Quick start

Install the command-line tools:

```bash
mamba install -c conda-forge -c bioconda -c esteinig cerebro
```

This installs several binaries. The ones you will normally use:

```bash
cerebro          --help  # stack configuration and deployment
cerebro-client   --help  # interaction with a deployed stack (login, upload, query, tables)
cerebro-pipeline --help  # pipeline output processing and support tools
cerebro-report   --help  # clinical report configuration and compilation
cerebro-ciqa     --help  # validation, quality assurance and the experimental diagnostic LLM
```

The following are internal components of a deployed stack and are **not intended for direct use**:
`cerebro-server`, `cerebro-fs`, `cerebro-watcher`, `cerebro-tower`, `cerebro-worker`.

> [!NOTE]
> `cerebro-ciqa` tracks the `main` branch of `Cerebro` rather than the released version, and pulls in
> the `META-GPT` library. See [AI-assisted diagnostic interpretation](#disclosure-ai-assisted-diagnostic-interpretation).

Authenticate against a stack. The token is read from `CEREBRO_API_TOKEN` by all subsequent commands:

```bash
export CEREBRO_API_TOKEN=$(cerebro-client --url http://localhost:8080 login -e admin@cerebro -p admin)
```

Convert pipeline outputs into a database model and upload it:

```bash
# Build a sample model from the quality-control and pathogen-detection outputs
cerebro-client create-pathogen \
  --quality   ${sample_id}.qc.json \
  --pathogen  ${sample_id}.pd.json \
  --taxonomy  ${db_directory}/taxonomy \
  --pipeline-config ${output_directory}/config.json \
  --run-id    ${run_id} \
  -o ${sample_id}.model.json

# Create the target database and project, then upload
cerebro-client --team CNS database create --name Production --description "Production data"
cerebro-client --team CNS --db Production project create --name RUN01 --description "Sequencing run 01"
cerebro-client --team CNS --db Production --project RUN01 upload-models --models *.model.json
```

> [!IMPORTANT]
> Uploads are limited to 100 MB per model in the default stack configuration. Only upload samples
> from the same pipeline run into a single project, workflow selection is not yet enforced in the
> interface, and re-using a sample identifier within a project will raise an error.

Retrieve summary tables:

```bash
cerebro-client --team CNS --db Production --project RUN01 download-models -o data_models --summary-table models.tsv
cerebro-client --team CNS --db Production --project RUN01 get-quality-control-table -o models.qc.tsv
cerebro-client --team CNS --db Production --project RUN01 get-pathogen-detection-table -o models.pd.tsv
```

Generate quality control summary plots. DNA and RNA libraries should be plotted separately using
their sample tags:

```bash
cerebro-ciqa plot-qc -i *__DNA*.qc.json -o qc_dna.svg -p pos_dna.tsv -t 4
cerebro-ciqa plot-qc -i *__RNA*.qc.json -o qc_rna.svg -p pos_rna.tsv -t 4
```

Using `*.qc.json` rather than the full `*.model.json` is considerably faster. The thresholds applied here are described in the preprint; defaults are in [`QcConfig::illumina_pe_sr`](cerebro/stack/pipeline/src/modules/quality.rs).

> [!IMPORTANT]
> Samples must follow the naming scheme `{SAMPLE_ID}__{NUCLEIC_ACID}__{SAMPLE_TAG}_{TAIL}` for
> control co-selection and DNA/RNA pairing to work. Identifiers must be anonymised and must never
> contain spaces, linkable identifiers, or the actual sample site or type.


## Clinical reporting

### Quick start

Reports are compiled from a JSON (or TOML) configuration file into PDF using `Typst` templates.

```bash
# Write out a blank configuration template
cerebro-report template -o report.json

# Fill in the template, then compile
cerebro-report compile -c report.json -o report.pdf
```

The template covers the report header and logo, legal disclosure and disclaimer text, the
authorisation block and signatures, the patient header, the reported result, and optional laboratory,
bioinformatics and audit appendices.

> [!WARNING]
> The report configuration file contains protected patient information (name, URN, date of birth,
> requesting clinician, specimen identifiers). Treat it as clinical data: keep controlled copies
> only, and delete working copies when the report is issued.

For clinical samples the result fields are filled in **manually** from the Bug Board determination,
and the report requires authorisation before issue. In the web application, reports are compiled
**in the browser** using the `cerebro-report-wasm` binary, so patient information in the header is
never transmitted to the server.


## Application stack

### Quick start

Requires `docker` with `docker compose`, and `git`.

```bash
# Generate a stack configuration and deployment directory
cerebro stack deploy \
  --name cerebro-prod \
  --outdir cerebro-prod \
  --config localhost-insecure \
  --git-url https://github.com/vidrl/cerebro

# Start the stack
cd cerebro-prod && docker compose up
```

Then open `http://localhost:8000`. In the `localhost-insecure` configuration the administrator
account is `admin@cerebro` with password `admin`.

> [!WARNING]
> `localhost-insecure` uses default passwords for the database and administrator accounts. It is for
> local evaluation and development only, never deploy it in a real-world setting. Available
> configuration templates are `localhost`, `localhost-insecure`, `localhost-fs` and `web`; use `web`
> with `Traefik` and TLS for any shared deployment.

Before data can be uploaded, create a team. This adds the current administrator to the team:

```bash
export CEREBRO_API_TOKEN=$(cerebro-client login -e admin@cerebro -p admin)
cerebro-client team create --name CNS --description "Central nervous system infections"
```

Further users, teams and team membership are managed in the application admin settings. For local
deployments the email verification flow is generally disabled, so the administrator must create and
verify users and set their passwords manually.

For a development stack with hot-reload of the Svelte application and a manual rebuild trigger for
the Rust backend:

```bash
cerebro stack deploy --name cerebro-dev --outdir cerebro-dev --config localhost-insecure --dev --trigger
export UID=$(id -u) && export GID=$(id -g)   # required for mounted host directories
cd cerebro-dev && docker compose up
```

## Cerebro API

### Quick start

The API (`cerebro-server`) is deployed as part of the application stack. It listens on port `8080` by default; the application container reaches it internally at `http://cerebro-api:8080`.

Check that a stack is reachable and that your credentials work:

```bash
# Unauthenticated status
cerebro-client --url http://localhost:8080 ping-status

# Authenticate; the token is used by all subsequent commands
export CEREBRO_API_TOKEN=$(cerebro-client --url http://localhost:8080 login -e admin@cerebro -p admin)

# Authenticated ping
cerebro-client --url http://localhost:8080 ping-server
```

All client commands accept `--url`, `--team`, `--db` and `--project` on the parent command, before
the subcommand.

Authentication uses RS256 JWTs with access and refresh tokens issued as `HttpOnly`, `Secure`,
`SameSite=Strict` cookies, with sessions tracked in `Redis`. Passwords are hashed with `Argon2`. To
generate a password hash for the administrator account when configuring a stack manually:

```bash
cerebro stack hash
```

Outbound email (user invitations, verification, password reset) requires an SMTP relay configured in
`SmtpConfig` (`cerebro/lib/model/src/api/config.rs`). This is optional, an administrator can create
accounts and set passwords without the registration flow.

## Cerebro FS

### Quick start

> [!NOTE]
> `CerebroFS` is deployed alongside the local stack configurations but is **not actively used** in the
> current release. Unless you are working on distributed storage integration, you can ignore it.
> Any paths supplied to `--fs-primary` and `--fs-secondary` at deployment will simply have file
> system directories created in them.

`CerebroFS` provides distributed sequence and analysis file storage over `SeaweedFS`, with files
registered against the Cerebro API so that production pipelines can stage inputs from storage rather
than from a local path. The `cerebro-fs` client supports upload, download, delete, list, and staging
of samples for production pipeline execution.

Its main current use is internal: the production pipeline calls `cerebro-fs stage` to resolve staged
input files at the start of a run unless the `--fastq` argument is provided (see Nextflow section).


## Databases and taxonomy

### Quick start

The default pathogen detection configuration uses the **`Cipher`** diagnostic database-an
amalgamation of archaeal and bacterial (GTDB), eukaryotic (EuPath, WormBase) and viral (ICTV)
reference genome collections with a grafted taxonomy. It supplies the indices for all classifiers in
the profiling module (`Kraken2`, `Metabuli`, `Ganon2`, `Sylph`, `KMCP`, `Bracken`), the alignment
references (`Vircov`, `minimap2`) and the `BLAST` databases used for assembly-based confirmation.

Taxonomic identifiers in pipeline outputs are normalised against an NCBI-schema taxonomy directory
containing `nodes.dmp` and `names.dmp`, supplied when building a sample model:

```bash
cerebro-client create-pathogen \
  --quality ${sample_id}.qc.json \
  --pathogen ${sample_id}.pd.json \
  --taxonomy ${db_directory}/taxonomy \
  -o ${sample_id}.model.json
```

Useful options:

- `--gtdb` collapses paraphyletic GTDB species into a single species (`_A`, `_B`, ...), which may reassign taxonomic
  identifiers. It is mainly used for testing, you should only use it for experiments
- `--strict` raises an error when a taxonomic identifier cannot be resolved, instead of skipping it

> [!NOTE]
> Custom database and index construction is handled by [`Cipher`](https://github.com/esteinig/cipher).
> It is a research tool, is not yet user-friendly, and is not ready for general deployment. Building
> a replacement database currently requires constructing indices for each classifier manually.

## Disclosure: AI-assisted diagnostic interpretation

Cerebro includes an experimental capability for large language model (LLM) assisted interpretation of metagenomic results. It is implemented in the `cerebro-ciqa` module (cerebro/stack/ciqa) via the `meta-gpt` crate, and combines a structured decision tree with a locally deployed, open-weight reasoning model (Qwen3, GGUF weights, default `qwen3-8b-q8-0`) to assign diagnoses and select pathogen candidates.

This feature is used through the `cerebro-ciqa` diagnose-local subcommand. It is not part of the Nextflow detection pipelines, and it is not invoked automatically anywhere in the clinical reporting path - no result reaches a report through this module unless a user explicitly runs it.

### Evaluation and its limits

The approach was evaluated in the preprint listed above and has not been peer reviewed.

Reported performance was obtained on a single-centre, non-representative cohort and does not support generalisation:

* One study, one site, one assay. All data derive from the META-GP study (Victoria, Australia, 2024–2025) using a single short-read Illumina protocol for sterile-site specimens (cerebrospinal and ocular fluid). No other specimen type, sample matrix, sequencing platform, laboratory or population was assessed
* Small, constructed validation set: validation comprised clinical samples, spike-ins and controls; it is not a consecutive, prospectively collected clinical series
* Headline figures are from a filtered subset. The reported sensitivity and specificity (94.4% / 95.4% without clinical notes; 97.2% / 100% with clinical notes) are for the subset above the experimental limit of detection (n = 79)
* The development cohort (n = 78) was heterogeneous and was not designed to estimate performance
* Not assessed: prospective use, high-biomass specimen types, other pathogens, other model families, or any deployment outside the evaluated configuration

### Reproducibility caveat

Outputs are stochastic and are sensitive to the model, quantisation, system prompt, and decision tree configuration. Changing any of these changes behaviour and invalidates the reported performance.

The `meta-gpt` dependency is currently pinned to a git branch (`main`) rather than a tag or commit. A new build made will not necessarily reproduce the system that was evaluated. Pin the dependency to a specific revision before using this module for anything you intend to rely on.

### Conditions of use

This feature must not be used as the sole or primary basis for a clinical diagnosis, a patient report, or any patient management decision. It is intended to support review by a qualified expert, and does not replace the expertise required to evaluate all possible pathogen detections (see development cohort in preprint).

Cerebro and its dependencies have no regulatory clearance or approval from the TGA, FDA, or any other authority. It is not an approved in vitro diagnostic and is not accredited for diagnostic reporting. Any diagnostic use requires local validation on your own specimens, protocol and population, under your laboratory's quality management system and accreditation requirements. Every output requires human expert review before it informs anything.

### Research code

Cerebro is a research project. It is not a production-ready or clinically validated codebase, and it should not be treated as one. The repository describes deployment in clinical and public health settings because that is the intended eventual purpose. It does not yet meet the standard that purpose requires. Users should be aware that, at present:

* The codebase has minimal automated test coverage, and no verification suite that would reliably catch an incorrect change to detection, filtering, or scoring logic
* Interfaces, output formats, and configuration are unstable and change without notice or migration path. Releases are not yet versioned
* No formal software life cycle process (e.g. IEC 62304), risk management file (ISO 14971), or design history is currently maintained for this repository
* The pipelines have been validated for research use on the specimen types described above only

Anyone considering clinical or accredited use is responsible for their own validation, verification, risk assessment, change control and regulatory compliance. Please open an issue or contact the maintainers (Ramachandran lab) before deploying Cerebro in a diagnostic setting.


## Status

The evaluation study for Cerebro and META-GPT is in preprint:

> Eike Steinig, Marcelina Krysiak, Kirti Deo, Andrew Duncan, Jacqueline Prestedge, Jeremy Barr, Jean Moselen, Sadid F. Khan, Janath A. Fernando, Ivana Savic, Bhargavi Yellapu, Ammar Aziz, Wytamma Wirth, Jessica Parry, Angela McDonald, Chhay Lim, Sharon Trevor, Benjamin Aw-Yeong, Georgia McCluskey, Michael Moso, Eddie Chan, Sonia L. La Vita, Penelope A. Bryant, Amy Crowe, Ramla Maalim, Diana Velasquez Reyes, Maryza Graham, Eloise Williams, Jason C. Kwong, Rachel Woolstencroft, Monica Slavin, Lyndell L Lim, Lachlan J.M. Coin, Leon Caly, Katherine Bond, Chuan Kok Lim, Timothy P. Stinear, Deborah A. Williamson, Prashanth S. Ramachandran - **Large language models enable consensus-level interpretation in metagenomic diagnostics** - medRxiv (2026) - [2026.07.29.26358751](https://doi.org/10.64898/2026.07.29.26358751)

Cerebro includes code for the viral enrichment branch of the pipeline used in:

> Michael A Moso, George Taiaroa, Eike Steinig, Madiyar Zhanduisenov, Grace Butel-Simoes, Ivana Savic, Mona L Taouk, Socheata Chea, Jean Moselen, Jacinta O’Keefe, Jacqueline Prestedge, Georgina L Pollock, Mohammad Khan, Katherine Soloczynskyj, Janath Fernando, Genevieve E Martin, Leon Caly, Ian G Barr, Thomas Tran, Julian Druce, Chuan K Lim, Deborah A Williamson - **Non-SARS-CoV-2 respiratory viral detection and whole genome sequencing from COVID-19 rapid antigen test devices: a laboratory evaluation study** - Lancet Microbe (2024) -[10.1016/S2666-5247(23)00375-0](https://doi.org/10.1016/S2666-5247(23)00375-0)