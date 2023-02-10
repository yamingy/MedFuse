MIMIC-IV Data Extraction
=========================

Here we modified the [codes](https://github.com/YerevaNN/mimic3-benchmarks/) for the publicly available Medical Information Mart for Intensive Care (MIMIC-IV) database ([paper](http://www.nature.com/articles/sdata201635), [website](http://mimic.physionet.org)). 

## Structure

       cd mimic4extract 
The `mimic3benchmark/scripts` directory contains scripts for creating the benchmark datasets.
The reading tools are in `mimic3benchmark/readers.py`.
All evaluation scripts are stored in the `mimic3benchmark/evaluation` directory.
The `mimic3models` directory contains the baselines models along with some helper tools.
Those tools include discretizers, normalizers and functions for computing metrics.


## Requirements

We do not provide the MIMIC-IV data itself. You must acquire the data yourself from https://mimic.physionet.org/. Specifically, download the CSVs. Otherwise, generally we make liberal use of the following packages:


## Building a benchmark data

    
2. The following command takes MIMIC-IV CSVs, generates one directory per `SUBJECT_ID` and writes ICU stay information to `data/{SUBJECT_ID}/stays.csv`, diagnoses to `data/{SUBJECT_ID}/diagnoses.csv`, and events to `data/{SUBJECT_ID}/events.csv`. This step might take around an hour.

       python -m mimic3benchmark.scripts.extract_subjects_iv {PATH TO MIMIC-IV CSVs} data/root/
   Output using MIMIC-IV v2.1:

       START:
              stay_ids: 73141
              hadm_ids: 66189
              subject_ids: 50934
       REMOVE ICU TRANSFERS:
              stay_ids: 67811
              hadm_ids: 61883
              subject_ids: 48261
       REMOVE MULTIPLE STAYS PER ADMIT:
              stay_ids: 56842
              hadm_ids: 56842
              subject_ids: 45140
       REMOVE PATIENTS AGE < 18:
              stay_ids: 56842
              hadm_ids: 56842
              subject_ids: 45140
       Breaking up stays by subjects: 100%|█████████████████████████████████████████████████████████████| 45140/45140 [03:04<00:00, 245.03it/s]
       Breaking up diagnoses by subjects: 100%|█████████████████████████████████████████████████████████| 45140/45140 [02:50<00:00, 264.42it/s]
       Processing OUTPUTEVENTS table:  95%|████████████████████████████████████████████████████▎  | 4234697/4457381 [03:23<00:10, 20788.48it/s]
       Processing CHARTEVENTS table:  95%|███████████████████████████████████████████████████████▎  | 314035266/329499788 [46:29<02:17, 112570.88it/s]
       Processing LABEVENTS table:  97%|██████████████████████████████████████████████████████████▉  | 118057948/122103667 [20:13<00:41, 97323.33it/s]

3. The following command attempts to fix some issues (ICU stay ID is missing) and removes the events that have missing information. About 80% of events remain after removing all suspicious rows (more information can be found in [`mimic3benchmark/scripts/more_on_validating_events.md`](mimic3benchmark/scripts/more_on_validating_events.md)).

       python -m mimic3benchmark.scripts.validate_events data/root/

4. The next command breaks up per-subject data into separate episodes (pertaining to ICU stays). Time series of events are stored in ```{SUBJECT_ID}/episode{#}_timeseries.csv``` (where # counts distinct episodes) while episode-level information (patient age, gender, ethnicity, height, weight) and outcomes (mortality, length of stay, diagnoses) are stores in ```{SUBJECT_ID}/episode{#}.csv```. This script requires two files, one that maps event ITEMIDs to clinical variables and another that defines valid ranges for clinical variables (for detecting outliers, etc.). **Outlier detection is disabled in the current version**.

       python -m mimic3benchmark.scripts.extract_episodes_from_subjects data/root/

5. The next command splits the whole dataset into training and testing sets. Note that the train/test split is the same of all tasks. 

       python -m mimic3benchmark.scripts.split_train_and_test data/root/
	
6. The following commands will generate task-specific datasets, which can later be used in models. These commands are independent, if you are going to work only on one benchmark task, you can run only the corresponding command.

       python -m mimic3benchmark.scripts.create_in_hospital_mortality data/root/ data/in-hospital-mortality/
       python -m mimic3benchmark.scripts.create_decompensation data/root/ data/decompensation/
       python -m mimic3benchmark.scripts.create_length_of_stay data/root/ data/length-of-stay/
       python -m mimic3benchmark.scripts.create_phenotyping data/root/ data/phenotyping/

After the above commands are done, there will be a directory `data/{task}` for each created benchmark task.
These directories have two sub-directories: `train` and `test`.
Each of them contains bunch of ICU stays and one file with name `listfile.csv`, which lists all samples in that particular set.
Each row of `listfile.csv` has the following form: `icu_stay, period_length, label(s)`.
A row specifies a sample for which the input is the collection of ICU event of `icu_stay` that occurred in the first `period_length` hours of the stay and the target is/are `label(s)`.
In in-hospital mortality prediction task `period_length` is always 48 hours, so it is not listed in corresponding listfiles.


### Train / validation split

Use the following command to extract validation set from the training set. This step is required for running the baseline models. Likewise the train/test split, the train/validation split is the same for all tasks.

       python -m mimic3models.split_train_val {dataset-directory}
       
`{dataset-directory}` can be either `data/in-hospital-mortality`, `data/phenotyping`.

