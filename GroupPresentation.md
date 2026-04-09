# Group Presentation


## Table of Contents
1. [fMRIprep](#mcdaniel)
2. [tedana](#nicole)
3. [Level 1 Analysis](#ranesh)

# fMRIPrep <a id='mcdaniel'></a>

## Introduction

This notebook documents the first stages of preprocessing for the OpenNeuro dataset `ds003507` using `fMRIPrep` in Neurodesk. The goal of this workflow is to demonstrate how to prepare one subject's BIDS-formatted data and launch preprocessing for a single participant. 

In this notebook, I begin the preprocessing workflow for the OpenNeuro dataset `ds003507`, titled *Audiovisual Valence Congruence*. The notebook is based on the course demo for `fMRIPrep` and is adapted for my selected dataset and subject. For this assignment, I focus only on the first three major steps: loading the required tools and libraries, preparing the data, and running `fMRIPrep` for one subject.

This step is important because preprocessing organizes and standardizes the raw MRI data before later statistical analysis. By running `fMRIPrep`, I am beginning the process of correcting, aligning, and preparing the anatomical and functional data in a way that will support later analysis in FSL and related neuroimaging tools.

## Workflow Overview

This notebook is organized into three main sections:

1. **Load software tools and import Python libraries**  
   In this section, I load the Neurodesk software tools required for preprocessing and import the Python libraries needed for file handling and notebook organization.

2. **Data preparation**  
   In this section, I create a working directory structure, install the dataset using DataLad, and download one full subject so that the anatomical, functional, and field map files are all available for preprocessing.

3. **Run fMRIPrep for one subject**  
   In this section, I launch `fMRIPrep` for `sub-01`, directing the outputs to a derivatives folder and using a scratch working directory for temporary processing files.

Dataset used: `ds003507`  
Selected subject: `sub-01`

## 1. Load software tools and import Python libraries

In this section, I load the software modules needed for the preprocessing workflow and import the Python packages used to manage paths, inspect files, and organize notebook outputs. This ensures that the environment is ready before preparing the dataset and launching `fMRIPrep`.

```python
import module
await module.load('fmriprep/25.2.5')
await module.load('fsl/6.0.7.16')
await module.list()
```

Expected output:

```python
['fmriprep/25.2.5', 'fsl/6.0.7.16']
```

```python
!pip install pandas ipyniivue
```

```python
from pathlib import Path
import os
import glob
import pandas as pd
from IPython.display import display, Markdown, Image

base_dir = Path.home() / "ds003507_fmriprep_example"
print(base_dir)
```

Expected output:

```python
/home/jovyan/ds003507_fmriprep_example
```

## 2. Data preparation

In this section, I prepare a local working directory for the preprocessing example. I create folders for the BIDS dataset, derivatives, and scratch files. I then install the OpenNeuro dataset `ds003507` using DataLad and recursively download `sub-01` so that all required anatomical, functional, and field map files are available for preprocessing.

We will keep everything for this example in one folder directly under the home directory:

`~/ds003507_fmriprep_example`

Inside that folder, we will make a few subdirectories:

- `bids/` for the OpenNeuro dataset  
- `derivatives/` for fMRIPrep output  
- `scratch/` for temporary working files

### Prepare folders and download one full subject

The recursive download of `sub-01` is important because `fMRIPrep` may require multiple file types for preprocessing, including anatomical images, functional runs, and field map images. Downloading the full subject ensures that the subject's data are locally available and organized in valid BIDS structure before preprocessing begins.

```bash
%%bash
set -e

EXAMPLE_DIR="$HOME/ds003507_fmriprep_example"

mkdir -p "$EXAMPLE_DIR"
cd "$EXAMPLE_DIR"

mkdir -p bids derivatives scratch

cd bids

# Install dataset skeleton if needed
if [ ! -d ds003507 ]; then
  datalad install https://github.com/OpenNeuroDatasets/ds003507.git
fi

cd ds003507

# Get the full subject recursively so fMRIPrep has anat, func, and fmap files
datalad get -r sub-01
```

Representative output:

```text
[INFO] Ensuring presence of Dataset(/home/jovyan/ds003507_fmriprep_example/bids/ds003507) to get /home/jovyan/ds003507_fmriprep_example/bids/ds003507/sub-01
```

### Confirm the subject files are present

For this example, I am preprocessing:

- **subject**: `01`

This dataset includes anatomical, functional, and fieldmap files for the subject, so downloading the full subject recursively is the simplest way to prepare for `fMRIPrep`.

```python
!tree -L 3 ~/ds003507_fmriprep_example/bids/ds003507/sub-01
```

## 3. Run fMRIPrep for one subject

In this section, I run `fMRIPrep` on `sub-01`. The preprocessing outputs are directed to the `derivatives/fmriprep` folder, and temporary working files are written to a scratch directory. For this class workflow, I use settings that are appropriate for a lighter demonstration run, including skipping BIDS validation in this environment and turning off FreeSurfer's full `recon-all` step.

The next cells run `fMRIPrep` on one participant and write the outputs under:

`~/ds003507_fmriprep_example/derivatives/fmriprep`

Notes:
- the FreeSurfer license file should exist at `~/.license`
- this command may take a long time
- `--skip-bids-validation` is used to avoid validator issues in this environment
- `--fs-no-reconall` keeps the run lighter for class purposes

```python
!ls -l ~/.license
```

### Run fMRIPrep for `sub-01`

The `fMRIPrep` command takes the BIDS dataset as input and preprocesses one selected subject. At a high level, this step prepares the anatomical and functional MRI data so they can be used in later analyses. The outputs generated by `fMRIPrep` typically include preprocessed BOLD images, confound regressors, and an HTML report summarizing the preprocessing workflow and image alignment quality.

```bash
%%bash
set -e

EXAMPLE_DIR="$HOME/ds003507_fmriprep_example"

sub=01
BIDS_DIR="$EXAMPLE_DIR/bids/ds003507"
OUT_DIR="$EXAMPLE_DIR/derivatives/fmriprep"
WORK_DIR="$EXAMPLE_DIR/scratch/fmriprep_work"
FS_LIC="$HOME/.license"

mkdir -p "$OUT_DIR" "$WORK_DIR"

export OMP_NUM_THREADS=1
export ITK_GLOBAL_DEFAULT_NUMBER_OF_THREADS=1

fmriprep \
  "$BIDS_DIR" \
  "$OUT_DIR" \
  participant \
  --participant-label "$sub" \
  --stop-on-first-crash \
  --skip-bids-validation \
  --fs-license-file "$FS_LIC" \
  --fs-no-reconall \
  --output-spaces MNI152NLin6Asym:res-2 \
  --nthreads 8 \
  --omp-nthreads 1 \
  --mem-mb 20000 \
  -w "$WORK_DIR"
```

Representative output begins like this:

```text
260409-01:17:54,810 nipype.workflow IMPORTANT:
     Running fMRIPrep version 25.2.5
```

### Find the main output files after fMRIPrep finishes

After `fMRIPrep` finishes, the derivatives folder should contain preprocessed functional images, confound files, and an HTML report for the subject. These outputs will be used in later steps of the class workflow to inspect preprocessing quality and support first-level analysis in FSL.

```python
!find ~/ds003507_fmriprep_example/derivatives/fmriprep -name "sub-01.html"
!find ~/ds003507_fmriprep_example/derivatives/fmriprep -name "*desc-preproc_bold.nii.gz"
!find ~/ds003507_fmriprep_example/derivatives/fmriprep -name "*desc-confounds_timeseries.tsv"
```

Representative output:

```text
/home/jovyan/ds003507_fmriprep_example/derivatives/fmriprep/sub-01.html
/home/jovyan/ds003507_fmriprep_example/derivatives/fmriprep/sub-01/func/sub-01_task-affect_run-2_space-MNI152NLin6Asym_res-2_desc-preproc_bold.nii.gz
/home/jovyan/ds003507_fmriprep_example/derivatives/fmriprep/sub-01/func/sub-01_task-affect_run-2_desc-confounds_timeseries.tsv
```

### Display the fMRIPrep HTML report
![unnamed](https://github.com/user-attachments/assets/2c475fde-f0e4-4289-bda4-0f571cc6bbc4)


## Summary

In this notebook, I completed the first three steps of the preprocessing workflow for the OpenNeuro dataset `ds003507` in Neurodesk. I loaded the required software tools and Python libraries, prepared the dataset by installing it with DataLad and downloading one full subject, and ran `fMRIPrep` for `sub-01`.

This notebook demonstrates the setup required to begin preprocessing a BIDS-formatted fMRI dataset. Even before later FEAT analysis, these steps are important because they establish the dataset structure, confirm subject-level data availability, and begin standardized preprocessing for downstream analysis.



### Running Tedana <a id='nicole'></a>

After fMRIPrep, we ran tedana to leverage our multi-echo fMRI acquisition.
Tedana uses ICA to distinguish BOLD signal from noise based on TE-dependence: true BOLD signal shows predictable T2* decay across echoes, while non-BOLD artifacts (motion, scanner drift, physiological noise) exhibit TE-independent patterns. This allows tedana to classify components as signal (accepted) or noise (rejected).

Script to run tedana
- This Runs tedana on all 4 runs for each subject using the 4 preprocessed echo files from fMRIPrep. Edit the SUBJECTS list at the top to specify subjects.

#### Bash script to run tedana
```
#!/bin/bash

# List of subjects 
SUBJECTS="302 303 304 305 306 307 308 309 310 311"

# Paths
DERIVATIVES_DIR="/zpool/olsonlab/active_drive/ljhoffman/pgt/derivatives"

# For loop through subjects
for SUBJECT in ${SUBJECTS}
do
  echo "Processing subject: ${SUBJECT}"
  
  FMRIPREP_DIR="${DERIVATIVES_DIR}/fmriprep/sub-${SUBJECT}"
  TEDANA_OUT="${DERIVATIVES_DIR}/tedana/sub-${SUBJECT}"
  mkdir -p ${TEDANA_OUT}/func
  
  # Process each run (1-4)
  for RUN in 1 2 3 4
  do
    RUN_FORMATTED=$(printf "%02d" ${RUN})
    
    echo "  Run ${RUN}..."
    
    # Run tedana for all 4 echoes
    tedana \
      -d \
        ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_echo-1_desc-preproc_bold.nii.gz \
        ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_echo-2_desc-preproc_bold.nii.gz \
        ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_echo-3_desc-preproc_bold.nii.gz \
        ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_echo-4_desc-preproc_bold.nii.gz \
      -e 13.8 29.7 45.6 61.5 \
      --mask ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_desc-brain_mask.nii.gz \
      --out-dir ${TEDANA_OUT}/func/run-${RUN_FORMATTED} \
      --prefix sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED} \
      --convention bids \
      --n-threads 16
  done
  
done

```

#### Run Tedana Script Command
```
bash ~ /tedana.sh
```








Great! Now that we have run tedana, we must pull the files we want to make confound files ADD LATER NICOLE
This is python script that combines confounds from fMRIprep and tedana. 











```
#!/usr/bin/env python
import os
import pandas as pd
from natsort import natsorted
import re

# Paths
base_dir = '/zpool/olsonlab/active_drive/ljhoffman/pgt'
tedana_dir = f'{base_dir}/derivatives/tedana/'
fmriprep_dir = f'{base_dir}/derivatives/fmriprep/'
output_dir = f'{base_dir}/derivatives/confounds/LSS'

# Find all tedana metrics files
metric_files = natsorted([
    os.path.join(root, f)
    for root, dirs, files in os.walk(tedana_dir)
    for f in files
    if f.endswith("desc-tedana_metrics.tsv")
])

# Process each run
for file in metric_files:
    fname = os.path.basename(file)
    run_dir = os.path.dirname(file)
    
    # Parse filename
    sub = re.search(r'(sub-\d+)', fname).group(1)
    task = re.search(r'_task-(.*?)_', fname).group(1)
    run = re.search(r'_run-(\d+)', fname).group(1)
    
    # Read files
    fmriprep = pd.read_csv(f"{fmriprep_dir}{sub}/func/{sub}_task-{task}_run-{run}_desc-confounds_timeseries.tsv", sep='\t')
    mixing = pd.read_csv(os.path.join(run_dir, f"{sub}_task-{task}_run-{run}_desc-ICA_mixing.tsv"), sep='\t')
    metrics = pd.read_csv(file, sep='\t')
    
    # Extract rejected ICA components
    rejected = metrics[metrics['classification'] == 'rejected']['Component']
    rejected_idx = rejected.str.replace('ICA_', '').astype(int)
    rejected_ica = mixing.iloc[:, rejected_idx]
    
    # Select fMRIPrep confounds
    confound_cols = ['trans_x', 'trans_y', 'trans_z', 'rot_x', 'rot_y', 'rot_z'] + \
                    [f'a_comp_cor_{i:02d}' for i in range(6)] + ['framewise_displacement']
    fmriprep_confounds = fmriprep[confound_cols].fillna(0)
    
    # Combine and save as a tsv file
    confounds = pd.concat([fmriprep_confounds, rejected_ica], axis=1)
    os.makedirs(f"{output_dir}/{sub}", exist_ok=True)
    confounds.to_csv(f"{output_dir}/{sub}/{sub}_task-{task}_run-{run}_desc-LSS_confounds.tsv", sep='\t', index=False)
```

Command to Run This Python Script
- activate your conda environment first
```
python ~/make_confounds_LSS.py
```

TSV Output example
<img width="963" height="179" alt="Screenshot 2026-04-09 at 11 33 32 AM" src="https://github.com/user-attachments/assets/fa2d692f-e590-43f3-898a-b01d42e07c85" />

    
### Running Level 1 analyses <a id='ranesh'></a>
Assuming that you created your confound files, and you ran fmriprep and you have your task event file (in 3 columns), we can run least squares single to extract a single beta map corresponding to each trial within the task event files.

Note that we have 4 runs of data, and each run has 3 trials for both the self and other condition, so we will pull a beta map for each trial and each run, such that we have the following file structure for our outputs:
```
sub-302/
  run-01/
    self_induction/
      sub-302_run-01_trial-1_self_induction_beta.nii.gz
      sub-302_run-01_trial-2_self_induction_beta.nii.gz
      sub-302_run-01_trial-3_self_induction_beta.nii.gz
    other_induction/
      sub-302_run-01_trial-1_other_induction_beta.nii.gz
      sub-302_run-01_trial-2_other_induction_beta.nii.gz
      sub-302_run-01_trial-3_other_induction_beta.nii.gz
  run-02/
    self_induction/
      sub-302_run-02_trial-4_self_induction_beta.nii.gz
```

Copy the code below and create a file titled LSS_beta_L1.py

```
import os
import sys
import numpy as np
import pandas as pd
from nilearn.glm.first_level import FirstLevelModel
import time

# Get subjects from command line
if len(sys.argv) > 1:
    subjects = sys.argv[1:]  # All arguments after script name
else:
    print("Usage: python3 test_lss_pgt.py sub-302 sub-303 sub-304 ...")
    sys.exit(1)

print(f"\n{'='*60}")
print(f"LSS Beta Extraction")
print(f"Processing {len(subjects)} subjects: {', '.join(subjects)}")
print(f"Started at: {time.strftime('%Y-%m-%d %H:%M:%S')}")
print(f"{'='*60}\n")

base_dir = "/zpool/olsonlab/active_drive/ljhoffman/pgt"
fmriprep_dir = f"{base_dir}/derivatives/fmriprep"
tedana_dir = f"{base_dir}/derivatives/tedana"
timing_dir = f"{base_dir}/sharepoint/time_data"
confounds_base_dir = f"{base_dir}/derivatives/confounds/LSS"
out_base_dir = f"{base_dir}/derivatives/RSA/lss_betas/v2/"

runs = ["01", "02", "03", "04"]
TR = 1.615

def read_events(txt_file, trial_type):
    arr = np.loadtxt(txt_file)
    if arr.ndim == 1:
        arr = arr[None, :]
    return pd.DataFrame({
        "onset": arr[:, 0],
        "duration": arr[:, 1],
        "trial_type": trial_type
    })

# Loop through subjects
for subj in subjects:
    print(f"\n{'#'*60}")
    print(f"# SUBJECT: {subj}")
    print(f"# Started: {time.strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'#'*60}\n")
    
    confounds_dir = f"{confounds_base_dir}/{subj}"
    out_dir = f"{out_base_dir}/{subj}"
    
    self_trial_num = 1
    other_trial_num = 1
    
    for run in runs:
        print(f"\n{'='*60}")
        print(f"Processing {subj} run-{run}")
        print(f"{'='*60}")
        
        bold_file = f"{fmriprep_dir}/{subj}/func/{subj}_task-PGT_run-{run}_space-MNI152NLin2009cAsym_res-2_desc-preproc_bold.nii.gz"
        mask_file = f"{fmriprep_dir}/{subj}/func/{subj}_task-PGT_run-{run}_space-MNI152NLin2009cAsym_res-2_desc-brain_mask.nii.gz"
        confounds_file = f"{confounds_dir}/{subj}_task-PGT_run-{run}_desc-LSS_confounds.tsv"
        
        if not all(os.path.exists(f) for f in [bold_file, mask_file, confounds_file]):
            print(f"WARNING: Missing files for run {run}, skipping")
            print(f"  BOLD: {os.path.exists(bold_file)}")
            print(f"  Mask: {os.path.exists(mask_file)}")
            print(f"  Confounds: {os.path.exists(confounds_file)}")
            continue

        self_file = f"{timing_dir}/{subj}/run-{run}/self_induction.txt"
        other_file = f"{timing_dir}/{subj}/run-{run}/other_induction.txt"

        self_events = read_events(self_file, "self_induction")
        other_events = read_events(other_file, "other_induction")
        events = pd.concat([self_events, other_events], ignore_index=True).sort_values("onset").reset_index(drop=True)
        
        print(f"Found {len(self_events)} self trials and {len(other_events)} other trials")

        confounds = pd.read_csv(confounds_file, sep="\t")
        print(f"Using {confounds.shape[1]} confound regressors")

        run_out = f"{out_dir}/run-{run}"
        self_out = f"{run_out}/self_induction"
        other_out = f"{run_out}/other_induction"
        os.makedirs(self_out, exist_ok=True)
        os.makedirs(other_out, exist_ok=True)

        for i in range(len(events)):
            trial = events.iloc[[i]].copy()
            others = events.drop(i).copy()
            cond = trial.iloc[0]["trial_type"]

            trial["trial_type"] = "trial_of_interest"
            others["trial_type"] = "other_trials"
            lss_events = pd.concat([trial, others], ignore_index=True)
            
            model = FirstLevelModel(
                t_r=TR,
                hrf_model="spm",
                drift_model="cosine",
                high_pass=0.008,
                smoothing_fwhm=None,
                mask_img=mask_file,
                signal_scaling=False,
                minimize_memory=False
            )

            model.fit(run_imgs=bold_file, events=lss_events, confounds=confounds)
            beta = model.compute_contrast("trial_of_interest", output_type="effect_size")

            if cond == "self_induction":
                out_file = f"{self_out}/{subj}_run-{run}_trial-{self_trial_num:02d}_self_induction_beta.nii.gz"
                self_trial_num += 1
            else:
                out_file = f"{other_out}/{subj}_run-{run}_trial-{other_trial_num:02d}_other_induction_beta.nii.gz"
                other_trial_num += 1

            beta.to_filename(out_file)
            print(f"  ✓ Trial {i+1}/{len(events)}: {os.path.basename(out_file)}")
    
    print(f"\n{'#'*60}")
    print(f"# {subj} COMPLETE!")
    print(f"# Generated {self_trial_num-1} self betas and {other_trial_num-1} other betas")
    print(f"# Finished: {time.strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"{'#'*60}\n")

print(f"\n{'='*60}")
print(f"ALL SUBJECTS COMPLETE!")
print(f"Finished at: {time.strftime('%Y-%m-%d %H:%M:%S')}")
print(f"{'='*60}\n")
```

Paste the code below to execute the script
```
python ~/LSS_beta_L1.py
```

Let's take a look at an example subject:
/zpool/olsonlab/active_drive/ljhoffman/pgt/derivatives/RSA/lss_betas/v2/sub-302/run-01/other_induction/sub-302_run-01_trial-01_other_induction_beta.nii.gz

<img width="1917" height="885" alt="image" src="https://github.com/user-attachments/assets/e4d3834f-1ff1-482a-8c9d-bcd84b40f49c" />

### Yay! we sucessfully ran level one, we can take these trial wise beta maps and then move onto further downstream analyses. This was done to run representational similarity analyses, but you can use this pipeline to get trial wise beta estimates for other pipelines as well. 




