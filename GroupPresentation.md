# Group Presentation


## Table of Contents
1. [fMRIprep](#Mcdaniel)
2. [tedana](#Nicole)
3. [Level 1 Analysis](#Ranesh)

### Running Tedana <a id='Nicole'></a>

After fMRIPrep, we ran tedana to leverage our multi-echo fMRI acquisition.
Tedana uses ICA to distinguish BOLD signal from noise based on TE-dependence: true BOLD signal shows predictable T2* decay across echoes, while non-BOLD artifacts (motion, scanner drift, physiological noise) exhibit TE-independent patterns. This allows tedana to classify components as signal (accepted) or noise (rejected).

Script to run tedana
- This Runs tedana on all 4 runs for each subject using the 4 preprocessed echo files from fMRIPrep. Edit the SUBJECTS list at the top to specify subjects.

#### Tedana Script
```
#!/bin/bash

# Run tedana for multiple subjects - all 4 runs each
# Usage: bash tedana.sh

# List all subjects to process
SUBJECTS="308"

echo "========================================="
echo "Running tedana for subjects: ${SUBJECTS}"
echo "Processing all 4 runs per subject"
echo "5-minute break between subjects"
echo "========================================="
echo ""

# Set base paths
DERIVATIVES_DIR="/zpool/olsonlab/active_drive/ljhoffman/pgt/derivatives"

# Loop through each subject
for SUBJECT in ${SUBJECTS}
do
  echo ""
  echo "========================================="
  echo "Processing subject: ${SUBJECT}"
  echo "========================================="
  echo ""
  
  FMRIPREP_DIR="${DERIVATIVES_DIR}/fmriprep/sub-${SUBJECT}"
  TEDANA_OUT="${DERIVATIVES_DIR}/tedana/sub-${SUBJECT}"
  
  # Create output directory
  mkdir -p ${TEDANA_OUT}/func
  
  # Check if fMRIPrep outputs exist
  if [ ! -d "${FMRIPREP_DIR}/func" ]; then
    echo "⚠️  WARNING: fMRIPrep outputs not found for subject ${SUBJECT}, skipping..."
    continue
  fi
  
  # Process each run
  for RUN in 4
  do
    RUN_FORMATTED=$(printf "%02d" ${RUN})
    
    echo ""
    echo "=== Subject ${SUBJECT} - Run ${RUN} ==="
    echo ""
    
    # Check if echo-1 exists
    ECHO1="${FMRIPREP_DIR}/func/sub-${SUBJECT}_task-PGT_run-${RUN_FORMATTED}_echo-1_desc-preproc_bold.nii.gz"
    
    if [ ! -f "${ECHO1}" ]; then
      echo "⚠️  WARNING: Run ${RUN} not found, skipping..."
      continue
    fi
    
    # Run tedana
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
    
    # Check if tedana succeeded
    if [ $? -eq 0 ]; then
      echo "✓ Run ${RUN} completed successfully"
    else
      echo "✗ Run ${RUN} FAILED"
      exit 1
    fi
  done
  
  echo ""
  echo "✓ Subject ${SUBJECT} completed (all 4 runs)"
  echo ""
  echo "Waiting 5 minutes before starting next subject..."
  sleep 300  # 5 minutes
  
done

echo ""
echo "========================================="
echo "✓ ALL SUBJECTS COMPLETED"
echo "========================================="
echo ""
echo "Denoised outputs in: ${DERIVATIVES_DIR}/tedana/"
```

#### Run Tedana Script Command
```
bash ~ /tedana.sh
```

Great! Now that we have run tedana, we must pull the files we want to make confound files ADD LATER NICOLE


```
#!/usr/bin/env python
"""
Create confounds files optimized for LSS beta extraction.
"""

import os
import pandas as pd
from natsort import natsorted
import re

# FIXED: Use absolute paths
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

print(f"Found {len(metric_files)} runs to process")
print("="*60)

processed = 0
skipped = 0

for file in metric_files:
    fname = os.path.basename(file)
    run_dir = os.path.dirname(file)
    
    # Extract BIDS entities
    sub_match = re.search(r'(sub-\d+)', fname)
    task_match = re.search(r'_task-(.*?)_', fname)
    run_match = re.search(r'_run-(\d+)', fname)
    
    if not all([sub_match, task_match, run_match]):
        print(f"Could not parse BIDS entities from {fname}, skipping.")
        skipped += 1
        continue
    
    sub = sub_match.group(1)
    task = task_match.group(1)
    run = run_match.group(1)
    
    print(f"\n{sub} run-{run} task-{task}")
    
    # fMRIPrep confounds path
    fmriprep_fname = (
        f"{fmriprep_dir}{sub}/func/"
        f"{sub}_task-{task}_run-{run}_desc-confounds_timeseries.tsv"
    )
    
    if not os.path.exists(fmriprep_fname):
        print(f"  WARNING: fMRIPrep confounds not found")
        skipped += 1
        continue
    
    # Tedana ICA files
    ica_mixing_fname = os.path.join(run_dir, f"{sub}_task-{task}_run-{run}_desc-ICA_mixing.tsv")
    tedana_metrics_fname = file
    
    if not os.path.exists(ica_mixing_fname):
        print(f"  WARNING: ICA mixing not found")
        skipped += 1
        continue
    
    # Read files
    try:
        fmriprep_confounds = pd.read_csv(fmriprep_fname, sep='\t')
        ICA_mixing = pd.read_csv(ica_mixing_fname, sep='\t')
        metrics = pd.read_csv(tedana_metrics_fname, sep='\t')
    except Exception as e:
        print(f"  ERROR reading files: {e}")
        skipped += 1
        continue
    
    # Extract rejected ICA component timeseries
    try:
        rejected_components = metrics.loc[metrics['classification'] == 'rejected', 'Component']
        rejected_indices = rejected_components.str.replace('ICA_', '', regex=False).astype(int)
        bad_components = ICA_mixing.iloc[:, rejected_indices]
        bad_components.columns = [f"rejected_ICA_{i:02d}" for i in range(len(bad_components.columns))]
    except Exception as e:
        print(f"  WARNING: Could not extract rejected components: {e}")
        bad_components = pd.DataFrame()
    
    # Select fMRIPrep confounds for LSS (NO cosine, NO NSS)
    motion = ['trans_x', 'trans_y', 'trans_z', 'rot_x', 'rot_y', 'rot_z']
    acompcor = [f'a_comp_cor_{i:02d}' for i in range(6)]
    fd = ['framewise_displacement']
    
    desired_cols = motion + acompcor + fd
    filter_col = [c for c in desired_cols if c in fmriprep_confounds.columns]
    selected_confounds = fmriprep_confounds[filter_col].fillna(0)
    
    # Combine fMRIPrep and tedana confounds
    confounds_df = pd.concat([selected_confounds, bad_components], axis=1)
    
    # Output directory
    outdir = f"{output_dir}/{sub}"
    os.makedirs(outdir, exist_ok=True)
    
    outfname = f"{outdir}/{sub}_task-{task}_run-{run}_desc-LSS_confounds.tsv"
    
    # Save WITH headers
    confounds_df.to_csv(outfname, index=False, header=True, sep='\t')
    
    print(f"  ✓ {confounds_df.shape[0]} timepoints × {confounds_df.shape[1]} regressors")
    processed += 1

print("\n" + "="*60)
print(f"DONE! Processed: {processed}, Skipped: {skipped}")
print(f"Output: {output_dir}/")
print("="*60)
```

Command
```
python ~/make_confounds_LSS.py
```


    
### Running Level 1 analyses <a id='Ranesh'></a>
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




