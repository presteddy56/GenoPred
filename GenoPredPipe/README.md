# Genotype-based Prediction Pipeline (GenoPredPipe)

This is a snakemake pipeline for running the GenoPred scripts. The pipeline currently identifies the ancestry of each individual, calculates ancestry-matched reference-projected principal components, and calculates polygenic scores using a range of methods standardised using an ancestry matched reference. Finally, the pipeline provides a report of the results either for each individual or the sample overall.

The pipeline's ability to calculate polygenic scores has been validated [here](https://opain.github.io/GenoPred/Determine_optimal_polygenic_scoring_approach_GenoPredPipe.html) in UK Biobank.

***

## Getting started

### Step 1

Clone the repository

```bash
git clone https://github.com/opain/GenoPred.git
cd GenoPred/GenoPredPipe
```

### Step 2

Install [Anaconda](https://conda.io/en/latest/miniconda.html).

**Linux:**
```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sh Miniconda3-latest-Linux-x86_64.sh
```

Install Python 3.8, Snakemake 5.32, and the basic project dependencies. Note. I am installing these packages in an environment called 'base', if you already have an environment called 'base', you may need to create a new environment to avoid conflicts.

```bash
conda activate base
conda install python=3.8
conda install -c conda-forge mamba
mamba install -c bioconda -c conda-forge snakemake-minimal==5.32.2
mamba install dropbox
mamba install pandas
```

Note. If ghostscript is not already installed on your system, you will need to install it. You can check this by typing 'ghostscript' into the terminal.

### Step 3

Prepare a snakemake profile for parallel computing. I have provided an example for users using a slurm scheduler called 'example_slurm_profile_config.yaml'. Slurm users should create a folder called 'slurm' in '$HOME/.config/snakemake', and then copy in the example_slurm_profile_config.yaml, renaming it to config.yaml. More information about profiles in snakemake can be found [here](https://snakemake.readthedocs.io/en/stable/executing/cli.html#profiles). 

***

## Running pipeline

### Example using test data

#### Step 1

Download test data.

```bash
wget -O test_data.tar.gz https://zenodo.org/record/6093584/files/test_data.tar.gz?download=1
tar -xf test_data.tar.gz
rm test_data.tar.gz
```

#### Step 2

Run full pipeline for all target samples.

```bash
snakemake --profile slurm --use-conda run_create_reports
```

Note. If you receive an error saying 'MissingOutputException', you should try adding '--latency-wait 20' to the snakemake command, which tells the pipeline to wait 20 seconds between steps, thereby allowing filesystem latency.

Note. Please be patient when running the pipeline for the first time. Expect the 'downloading and installing remote packages' to take ~1 hour. It has to create the conda environment in first instance, which involves installing python and R and many packages. Expect this to take ~1 hour.

***

## Using your own data

You must specify a file listing target samples (target_list) and a file listing of GWAS summary statistics (gwas_list) for the pipeline to use. The location of these files should be specified in the config.yaml file. The column names for the gwas_list and target_list files should be as follows:

***

### target_list (required)

- name: Name of the target sample
- path: File path to the target genotype data. For type '23andMe', provide full file name either zipped (.zip) or uncompressed (.txt). For types 'samp_imp_plink1', 'samp_imp_bgen', and 'samp_imp_vcf', per-chromosome genotype data should be provided with the following filename format: \<prefix>.chr\<1-22>.\<.bed/.bim/.fam/.bgen/.vcf.gz>. If type 'samp_imp_bgen', the sample file should be called \<prefix>.sample, and each .bgen file should have a corresponding .bgi file.
- type: Either '23andMe', 'samp_imp_plink1', 'samp_imp_bgen', or 'samp_imp_vcf'. '23andMe' = 23andMe formatted data for an individual. 'samp_imp_plink1' = Preimputed PLINK1 binary format data (.bed/.bim/.fam) for a group of individuals. 'samp_imp_bgen' = Preimputed Oxford format data (.bgen/.sample) for a group of individuals. 'samp_imp_vcf' = Preimputed gzipped VCF format data (.vcf.gz) for a group of individuals.
- output: The desired output directory for target sample. The <name> of the target sample will automically be appended to the end. i.e. when output='test' and name='Joe_Bloggs', the output will written in a folder called 'test/Joe_Bloggs'.
- indiv_report: Either 'T' or 'F'. 'T' = Individual-specific ancestry and polygenic score reports will be created. Use with caution if target sample contains many individuals, as it will create an .html report for each individual.

Note. This file should be space delimited.

***

### gwas_list (required)

- name: Short name for the GWAS
- path: File path to the GWAS summary statistics (uncompressed or gzipped)
- population: The super population that the GWAS was performed in (AFR/AMR/EAS/EUR/SAS)
- sampling: The proportion of the GWAS sample that were cases (if binary, otherwise NA)
- prevalence: The population prevelance of the phenotype (if binary, otherwise NA)
- mean: The phenotype mean in the general population (if continuous, otherwise NA)
- sd: The phenotype sd in the general population (if continuous, otherwise NA)
- label: A human readable name for the GWAS phenotype. Wrap in quotes if multiple words. For example, "Body Mass Index".

Note. This file should be space delimited.

#### GWAS summary statistic format

The following column names are expected in the GWAS summary statistics files:

- SNP: RSID for variant (required)
- A1: Allele 1 (effect allele) (required)
- A2: Allele 2 (required)
- P: P-value of association (required)
- OR: Odds ratio effect size (required if binary)
- BETA: BETA effect size (required if continuous)
- SE: Standard error of log(OR) or BETA (required)
- N: Total sample size (required)
- FREQ: Allele frequency in GWAS sample (optional)
- INFO: Imputation quality (optional)

***

### score_list (required)

It is also possible to provide a list of external created score files (score_list) to be used for polygenic scoring. The score_list should have the same format as gwas_list, except the path column should indicate the location of the score file. Caution: GenoPredPipe assumes the externally provided score files are restricted and harmonised to the HapMap3 SNP-list.

If no externally created score files are present, please still provide a file containing the correct header.

Note. This file should be space delimited.

#### Score file format

The following column names are expected in the score files:

- SNP: RSID for variant (required)
- A1: Allele 1 (effect allele) (required)

Then one or more columns indicating the effect size of each variant, for example:

```
SNP         A1  SCORE_phi_1e-06   SCORE_phi_1e-04
rs3131972   A   4.502864e-07      -0.0002017462
rs3131969   A   1.48949e-05       8.115991e-06 
rs1048488   C   -1.371439e-05     4.194312e-05 
``` 

***

## Running parts of the pipeline

It is possible to only run specific parts of the pipeline for all target samples. For example:

```bash
# Harmonise with the reference only
snakemake --profile slurm --use-conda run_harmonise_target_with_ref

# Create ancestry report only
snakemake --profile slurm --use-conda run_create_ancestry_reports

# Calculate reference-projected principal components only
snakemake --profile slurm --use-conda run_target_pc_all

# Calculate pT+clump polygenic scores only
snakemake --profile slurm --use-conda run_target_prs_pt_clump_all_name

# Calculate DBSLMM polygenic scores only
snakemake --profile slurm --use-conda run_target_prs_dbslmm_all_name

```

Furthermore, you can request specific outputs for specific target samples. For example:

```bash
# Run full pipeline for specific target sample. 
# Replace <name> with name of target sample in the target_list file.
snakemake --profile slurm --use-conda resources/data/target_checks/<name>/create_sample_report.done

# Calculate pT+clump polygenic scores for specific population, in a specific target sample, and specific GWAS. 
# Replace <name> with name of a target sample in the target_list file. 
# Replace <gwas> with the name of a gwas in the gwas_list file.
# Replace <population> with the super population ancestry code (e.g. EUR, EAS, AMR, AFR, SAS).
snakemake --profile slurm --use-conda resources/data/target_checks/<name>/target_prs_pt_clump_<population>_<gwas>.done


# To generate polygenic scores using other methods, replace pt_clump in the previous command to either dbslmm, prscs, lassosum, sbayesr, ldpred2, megaprs, or external. For example, to calculate megaprs polygenic scores based on the BODY04 GWAS in the European (EUR) subset of the target sample called example_plink:
snakemake --profile slurm --use-conda resources/data/target_checks/example_plink1/target_prs_megaprs_EUR_BODY04.done

# Look inside the rules/GenoPredPipe.smk file to see all available rules.

```

***

## Output

Potentially useful GenoPredPipe outputs can be found in the following locations:

- 1KG Phase 3 reference data restricted to HapMap3 variants in PLINK1 binary format (.bed/.bim/.fam)
  - Per chromosome: resources/data/1kg/1KGPhase3.w_hm3.chr\<chr>.\<bed/bim/fam>
  - Genome-wide: resources/data/1kg/1KGPhase3.w_hm3.GW.\<.bed/.bim/.fam>
  - Super population and population keep files: resources/data/1kg/keep_files/\<pop>_sample.keep
  - Allele frequency files split by population: resources/data/1kg/freq_files/\<pop>/1KGPhase3.w_hm3.\<pop>.chr\<chr>.frq



- Quality controlled and formatted GWAS summary statistics: resources/data/gwas_sumstat/\<gwas>/\<gwas>.cleaned.gz
- Lassosum pseudovalidation results: resources/data/1kg/prs_pseudoval/\<gwas>
- Projected principal component .score files: resources/data/1kg/prs_score_files/\<method>/\<gwas>
- Polygenic score .score files: resources/data/1kg/prs_score_files/\<method>/\<gwas>



- Target sample outputs (\<output> corresponds to the output specified in the target_list file)
  - Imputed genotype data (if 23andMe input): \<output>/\<name>/\<name>.\<gen/sample>
  - Genotype data restricted to HapMap3 SNPs and harmonised with reference: \<output>/\<name>/\<name>.1KGPhase3.w_hm3.chr\<chr>.\<bed/bim/fam>
  - Super population ancestry results: \<output>/\<name>/ancestry
  - Within super population ancestry results: \<output>/\<name>/ancestry/ancestry_\<population>
  - Projected principal components: \<output>/\<name>/projected_pcs/\<population>
  - Polygenic scores: \<output>/\<name>/prs/\<population>/\<method>/\<gwas>
  - Individual-level report: \<output>/\<name>/reports
  - Sample-level report: \<output>/\<name>/reports

***

## Troubleshooting

If using the --profile flag, the log files will be saved in the GenoPredPipe/logs folder. If running interactively (i.e. -j1), the error should be printed on the screen.

***

Please contact Oliver Pain (oliver.pain@kcl.ac.uk) for any questions or comments.

***
