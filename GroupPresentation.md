# Group Presentation

Using This Pipeline with Your Own Data:
This pipeline can be adapted for any multi-echo fMRI dataset. To use these scripts with your own data, you'll need to update file paths, subject IDs, task names, and parameters specific to your acquisition.What you'll need to edit:

- File paths: Update all directory paths to point to your data
- Subject IDs: Replace our subject numbers (302-311) with yours
- Task name: Change task-PGT to your task identifier
- Echo times: Update the -e flag in tedana to match your TEs
- Condition names: Replace self_induction and other_induction with your conditions
- Acquisition parameters: Update TR, number of runs, smoothing preferences
- Throughout the scripts below, look for # EDIT THIS comments indicating what needs to be customized for your dataset. The core logic remains the same—only the specifics of your data need to change.

Note: We do not create timing files in this script. You must prepare your own 3-column timing files before running LSS beta extraction.


## Table of Contents
1. [fMRIprep](#mcdaniel)
2. [tedana](#nicole)
3. [Level 1 Analysis](#ranesh)

# fMRIPrep <a id='mcdaniel'></a>

## Introduction

This notebook documents the first stages of preprocessing for the OpenNeuro dataset `ds003507` using `fMRIPrep`. The goal of this workflow is to demonstrate how to prepare one subject's BIDS-formatted data and launch preprocessing for a single participant. 

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










#
# Running Tedana <a id='nicole'></a>

After fMRIPrep, we ran tedana to leverage our multi-echo fMRI acquisition.
Tedana uses ICA to distinguish BOLD signal from noise based on TE-dependence: true BOLD signal shows predictable T2* decay across echoes, while non-BOLD artifacts (motion, scanner drift, physiological noise) exhibit TE-independent patterns. This allows tedana to classify components as signal (accepted) or noise (rejected).

Script to run tedana
- This Runs tedana on all 4 runs for each subject using the 4 preprocessed echo files from fMRIPrep. Edit the SUBJECTS list at the top to specify subjects.

#### Bash script to run tedana

#### Copy the code below, replace with paths to your data, and create a file titled tedana_script.sh
```
#!/bin/bash

# EDIT THIS: Your subject ID (just the number, not "sub-")
SUBJECT="304"

# EDIT THIS: Path to your derivatives directory
DERIVATIVES_DIR="/path/to/your/derivatives"

FMRIPREP_DIR="${DERIVATIVES_DIR}/fmriprep/sub-${SUBJECT}"
TEDANA_OUT="${DERIVATIVES_DIR}/tedana/sub-${SUBJECT}"
mkdir -p ${TEDANA_OUT}/func

# EDIT THIS: Adjust to your number of runs (e.g., 1 2 3 for 3 runs)
for RUN in 1 2 3 4
do
  RUN_FORMATTED=$(printf "%02d" ${RUN})
  
  echo "Processing run ${RUN}..."
  
  # EDIT THIS: Replace "task-PGT" with your task name
  # EDIT THIS: Update echo times (-e flag) to match your acquisition
  tedana \
    -d \
      ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED}_echo-1_desc-preproc_bold.nii.gz \
      ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED}_echo-2_desc-preproc_bold.nii.gz \
      ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED}_echo-3_desc-preproc_bold.nii.gz \
      ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED}_echo-4_desc-preproc_bold.nii.gz \
    -e 13.8 29.7 45.6 61.5 \
    --mask ${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED}_desc-brain_mask.nii.gz \
    --out-dir ${TEDANA_OUT}/func/run-${RUN_FORMATTED} \
    --prefix sub-${SUBJECT}_task-YOUR_TASK_run-${RUN_FORMATTED} \
    --convention bids \
    --n-threads 16
done

echo "Subject ${SUBJECT} complete!"
```

### How to run?
Command to Run Bash Script
```
bash /path/to/your/tedana_script.sh
```

#### Tedana HTML report for 1 run of 1 sub

<img width="914" height="515" alt="Screenshot 2026-04-09 at 11 55 09 AM" src="https://github.com/user-attachments/assets/6d2bba34-6b95-40b2-9bca-6f2aa4e5e52f" />
Tedana ICA component classification showing accepted BOLD components (green) separated from rejected noise components (red) based on TE-dependence.

#
# Make Combined Confounds File
Great! Now that we have run tedana and fMRIprep, we need to prepare confound files for our first-level analysis. fMRIPrep and tedana create separate output files, so need to write a script to combine them. 


### Confounds from fMRIPrep:
1. Motion Parameters:
trans_x, trans_y, trans_z - Translation (mm) in X, Y, Z directions
rot_x, rot_y, rot_z - Rotation (radians) around X, Y, Z axes

2. aCompCor:
a_comp_cor_00 through a_comp_cor_05 - First 6 anatomical CompCor components
Extracted from CSF/white matter masks. Captures physiological noise (cardiac, respiratory)

3. Framewise Displacement:
framewise_displacement - Frame-to-frame motion metric (Power 2012)
Sum of absolute displacement across all 6 motion parameters

### Confounds from Tedana:
4. Rejected ICA Components:
rejected_ICA_00, rejected_ICA_01, rejected_ICA_02, etc.
ICA components tedana classified as non-BOLD noise

This may capture capture:
- Multi-echo specific artifacts
- Scanner drift
- Residual motion artifacts
- Susceptibility distortions



#### This is python script that combines confounds from fMRIprep and tedana. 

#### Copy the code below, replace with paths to your data, and create a file titled make_confounds.py

Install required packages (if needed):
```
pip install pandas natsort
```


```
#!/usr/bin/env python
import os
import pandas as pd
from natsort import natsorted
import re

# EDIT THIS: Path to your project directory
base_dir = '/path/to/your/project'
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

print(f"Found {len(metric_files)} runs to process\n")

# Process each run
for file in metric_files:
    fname = os.path.basename(file)
    run_dir = os.path.dirname(file)
    
    # Parse filename (assumes BIDS naming: sub-XXX_task-XXX_run-XX)
    sub = re.search(r'(sub-\d+)', fname).group(1)
    task = re.search(r'_task-(.*?)_', fname).group(1)
    run = re.search(r'_run-(\d+)', fname).group(1)
    
    print(f"Processing {sub} run-{run} task-{task}")
    
    # Read files
    fmriprep_file = f"{fmriprep_dir}{sub}/func/{sub}_task-{task}_run-{run}_desc-confounds_timeseries.tsv"
    mixing_file = os.path.join(run_dir, f"{sub}_task-{task}_run-{run}_desc-ICA_mixing.tsv")
    metrics_file = file
    
    fmriprep = pd.read_csv(fmriprep_file, sep='\t')
    mixing = pd.read_csv(mixing_file, sep='\t')
    metrics = pd.read_csv(metrics_file, sep='\t')
    
    # Extract rejected ICA components
    rejected = metrics[metrics['classification'] == 'rejected']['Component']
    rejected_idx = rejected.str.replace('ICA_', '').astype(int)
    rejected_ica = mixing.iloc[:, rejected_idx]
    
    # Select fMRIPrep confounds
    # EDIT THIS: Adjust number of aCompCor components if needed (we use 6)
    confound_cols = ['trans_x', 'trans_y', 'trans_z', 'rot_x', 'rot_y', 'rot_z'] + \
                    [f'a_comp_cor_{i:02d}' for i in range(6)] + \
                    ['framewise_displacement']
    
    fmriprep_confounds = fmriprep[confound_cols].fillna(0)
    
    # Combine and save
    confounds = pd.concat([fmriprep_confounds, rejected_ica], axis=1)
    os.makedirs(f"{output_dir}/{sub}", exist_ok=True)
    
    output_file = f"{output_dir}/{sub}/{sub}_task-{task}_run-{run}_desc-LSS_confounds.tsv"
    confounds.to_csv(output_file, sep='\t', index=False)
    
    print(f"  ✓ Saved: {confounds.shape[0]} timepoints × {confounds.shape[1]} regressors\n")

print("Done!")
```

### How to run?
Command to Run This Python Script
- activate your conda environment first
```
python /path/to/your/make_confounds.py
```

TSV Output example
<img width="963" height="179" alt="Screenshot 2026-04-09 at 11 33 32 AM" src="https://github.com/user-attachments/assets/fa2d692f-e590-43f3-898a-b01d42e07c85" />










#
# Running Level 1 analyses <a id='ranesh'></a>
Assuming that you created your confound files, and you ran fmriprep and you have your task event file (in 3 columns), we can run least squares single to extract a single beta map corresponding to each trial within the task event files.

Note that we have 4 runs of data, and each run has 3 trials for both the self and other condition, so we will pull a beta map for each trial and each run, such that we have the following file structure for our outputs:

#### Copy the code below, replace with paths to your data, and create a file titled LSS_beta_L1.py


Install required packages:
```
pip install numpy pandas nilearn nibabel
```


```
import os
import sys
import numpy as np
import pandas as pd
from nilearn.glm.first_level import FirstLevelModel
import time

# Get subjects from command line
if len(sys.argv) > 1:
    subjects = sys.argv[1:]
else:
    print("Usage: python3 lss_beta_extraction.py sub-302 sub-303 ...")
    sys.exit(1)

print(f"\nLSS Beta Extraction")
print(f"Processing {len(subjects)} subjects: {', '.join(subjects)}\n")

# EDIT THESE PATHS
base_dir = "/path/to/your/project"
fmriprep_dir = f"{base_dir}/derivatives/fmriprep"
timing_dir = f"{base_dir}/timing_files"  # Path to your timing files
confounds_dir = f"{base_dir}/derivatives/confounds/LSS"
out_dir = f"{base_dir}/derivatives/lss_betas"

# EDIT THESE: Your runs and TR
runs = ["01", "02", "03", "04"]  # Adjust to your number of runs
TR = 1.615  # Your repetition time in seconds

def read_events(txt_file, trial_type):
    """Read timing file (3 columns: onset, duration, trial_type)"""
    arr = np.loadtxt(txt_file)
    if arr.ndim == 1:
        arr = arr[None, :]
    return pd.DataFrame({
        "onset": arr[:, 0],
        "duration": arr[:, 1],
        "trial_type": trial_type
    })

# Process each subject
for subj in subjects:
    print(f"\n{'='*60}")
    print(f"Processing {subj}")
    print(f"{'='*60}\n")
    
    subj_confounds_dir = f"{confounds_dir}/{subj}"
    subj_out_dir = f"{out_dir}/{subj}"
    
    # Track trial numbers across runs
    condition1_trial_num = 1  # EDIT: Replace with your condition name
    condition2_trial_num = 1  # EDIT: Replace with your condition name
    
    for run in runs:
        print(f"Processing run-{run}")
        
        # EDIT THIS: Replace "task-PGT" with your task name
        bold_file = f"{fmriprep_dir}/{subj}/func/{subj}_task-YOUR_TASK_run-{run}_space-MNI152NLin2009cAsym_res-2_desc-preproc_bold.nii.gz"
        mask_file = f"{fmriprep_dir}/{subj}/func/{subj}_task-YOUR_TASK_run-{run}_space-MNI152NLin2009cAsym_res-2_desc-brain_mask.nii.gz"
        confounds_file = f"{subj_confounds_dir}/{subj}_task-YOUR_TASK_run-{run}_desc-LSS_confounds.tsv"
        
        # Check files exist
        if not all(os.path.exists(f) for f in [bold_file, mask_file, confounds_file]):
            print(f"  WARNING: Missing files for run {run}, skipping")
            continue
        
        # EDIT THIS: Paths to your timing files
        # Format: /path/to/timing/sub-XXX/run-XX/condition_name.txt
        condition1_file = f"{timing_dir}/{subj}/run-{run}/condition1.txt"
        condition2_file = f"{timing_dir}/{subj}/run-{run}/condition2.txt"
        
        # Read timing files
        condition1_events = read_events(condition1_file, "condition1")
        condition2_events = read_events(condition2_file, "condition2")
        events = pd.concat([condition1_events, condition2_events], ignore_index=True).sort_values("onset").reset_index(drop=True)
        
        print(f"  Found {len(condition1_events)} condition1 trials and {len(condition2_events)} condition2 trials")
        
        # Read confounds
        confounds = pd.read_csv(confounds_file, sep="\t")
        print(f"  Using {confounds.shape[1]} confound regressors")
        
        # Create output directories
        run_out = f"{subj_out_dir}/run-{run}"
        condition1_out = f"{run_out}/condition1"
        condition2_out = f"{run_out}/condition2"
        os.makedirs(condition1_out, exist_ok=True)
        os.makedirs(condition2_out, exist_ok=True)
        
        # LSS: Fit separate GLM for each trial
        for i in range(len(events)):
            trial = events.iloc[[i]].copy()
            others = events.drop(i).copy()
            cond = trial.iloc[0]["trial_type"]
            
            # Relabel for LSS
            trial["trial_type"] = "trial_of_interest"
            others["trial_type"] = "other_trials"
            lss_events = pd.concat([trial, others], ignore_index=True)
            
            # Fit GLM
            model = FirstLevelModel(
                t_r=TR,
                hrf_model="spm",  # EDIT: Can use "glover", "spm", "fir", etc.
                drift_model="cosine",
                high_pass=0.008,  # EDIT: Adjust if needed (1/128 Hz default)
                smoothing_fwhm=None,  # EDIT: Add smoothing if desired (e.g., 5)
                mask_img=mask_file,
                signal_scaling=False,
                minimize_memory=False
            )
            
            model.fit(run_imgs=bold_file, events=lss_events, confounds=confounds)
            beta = model.compute_contrast("trial_of_interest", output_type="effect_size")
            
            # Save beta map
            if cond == "condition1":
                out_file = f"{condition1_out}/{subj}_run-{run}_trial-{condition1_trial_num:02d}_condition1_beta.nii.gz"
                condition1_trial_num += 1
            else:
                out_file = f"{condition2_out}/{subj}_run-{run}_trial-{condition2_trial_num:02d}_condition2_beta.nii.gz"
                condition2_trial_num += 1
            
            beta.to_filename(out_file)
            print(f"  ✓ Trial {i+1}/{len(events)}")
    
    print(f"\n{subj} COMPLETE!")
    print(f"Generated {condition1_trial_num-1} condition1 betas and {condition2_trial_num-1} condition2 betas\n")

print("ALL SUBJECTS COMPLETE!")
```
### How to run?
Command to Run This Python Script
- activate your conda environment first
Paste the code below to execute the script. Run for 1 or multiple subjects
```
python lss_beta_extraction.py sub-## sub-##
```

Expected output:
```
derivatives/lss_betas/
└── sub-302/
    └── run-01/
        ├── condition1/
        │   ├── sub-302_run-01_trial-01_condition1_beta.nii.gz
        │   ├── sub-302_run-01_trial-02_condition1_beta.nii.gz
        │   └── ...
        └── condition2/
            ├── sub-302_run-01_trial-01_condition2_beta.nii.gz
            └── ...
```

Let's take a look at an example subject:
/zpool/olsonlab/active_drive/ljhoffman/pgt/derivatives/RSA/lss_betas/v2/sub-302/run-01/other_induction/sub-302_run-01_trial-01_other_induction_beta.nii.gz

<img width="1917" height="885" alt="image" src="https://github.com/user-attachments/assets/e4d3834f-1ff1-482a-8c9d-bcd84b40f49c" />

### Yay! we sucessfully ran level one, we can take these trial wise beta maps and then move onto further downstream analyses. This was done to run representational similarity analyses, but you can use this pipeline to get trial wise beta estimates for other pipelines as well. 




