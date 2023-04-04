# Workflow

In what follows, `/nfs/turbo/lsa-lukehyde/MTwiNS/mri/pipelines` will be
called `$pipes` and `$pipes/pipeline-resting-L1` will be called `$L1`.

We moved the `$L1/data` folder to `$L1/data.hold` so we can run tests
using the exact paths that were used before.

For testing, we are only using the `nophsyio` data.  The test subjects
chosen were: 6106t2 (mb), 6418t2 (sp), sub-683t2 (mb), sub-737t1 (mb),
sub-871t2 (sp).

According to the video, the preprocessing scripts must be run first.
Those are the Snakemake pipelines for `pipeline-resting-preproc` and
`pipeline-task2rest`.

Quality checks get done here by undergraduates.  As of Mar 31, 2022,
we do not know what those are.  We suspect that this may only be the
reorientation scheme that is used primarily for the ENIGMA project.

Then run `$pipes/pipeline-resting-preproc/bin/make_symlinks` and
`$pipes/pipeline-task2rest/bin/make_symlinks` (check that those
do their own type of data).  Those will create the folders under
`$L1/data/nophysio/preprocessed` for each subject, e.g.,

```
ls data/nophysio/preprocessed/
sub-6106t2/  sub-6418t2/  sub-683t2/  sub-737t1/  sub-871t2/
```

To create test data, use `rsync` to copy data from the `data.hold`
directory into the `data` directory using `rsync` from the
`$L1/data.hold/nophysio/preprocessed/sub-*` folders, which takes the place
of running the preprocessing steps listed above.

```
$ for sub in 6106t2 6418t2 683t2 737t1 871t2 ; do
    rsync -av data.hold/nophysio/preprocessed/sub-$sub
    data/nophysio/preprocessed
done
```

## batch_make_project.sh

This script takes arguments `-s [sessionID]`, `-d [data directory]`,
`-c [conn directory]`, `--cluster` (if running on GL).  For example,

```
$ $L1/bin/batch_make_project.sh -s 1 -d data/nophysio/preprocessed \
    -c data/nophysio/conn
```

The `batch_make_project.sh` scriipt indiscrimately assumes that all
directories in `data/nophysio/preprocessed/sub-*/ses-${ses}` contain valid
data.  The script then looks for a bold nifti file and sets the TR to 0.8
if the bold file contains `mb` and to `2` if it contains `sp` (multiband
vs spiral).  It then branches depending on whether it runs on the cluster.
For non-cluster jobs, it runs `$L1/bin/conn_make_project.sh -i ${sub}
-s ${ses} -d ${data_dir} -c ${conn_dir} --tr ${tr}`, where the variables
that are not the subject ID are those passed to  `batch_make_project.sh`.

Once that is done, then make the combination directories by running
`$L1/bin/single_subject_make_project.sh`, which is a hacked version of
`$L1/bin/batch_make_project.sh` with the subject ID hard coded inside it.
Change the subject in the script for each subject run.  Results for
each subject are in `$L1/make_project_[ID].log`.

## single_subject_make_project.sh

This script creates the individual conn projects under
`$L1/data/nophysio/conn/sub-*` as `batch_make_project.sh` would
using the same commands as are in `$L1/bin/conn_make_project.sh`
except that instead of running the pre-compiled `conn` command
from the Singularity container, it runs `matlab`, adds the paths
to SPM12 and CONN, then runs `conn_make_project()` as

```
conn_make_project('sub','${sub}', 'ses','${ses}',
    'conn_dir','${conn_dir}', 'data_dir','${data_dir}',
    'repetition_time','${tr}')
```

## conn_preprocess_one_sub.sh

This script sets the subject ID as `$sub`, it adds SPM12 and CONN to the
`MATLABPATH` variable so they are available in Matlab.  It then sets
`pipe_dir` to the root directory of the pipeline (`topDir` in some), i.e.,
`/nfs/turbo/lsa-lukehyde/MTwiNS/mri/pipelines/pipeline-resting-L1`, and
sets the `conn_dir` as a relative path.

It then loops over the found directories for each combination using

```
for combo in $(/bin/ls -d ${conn_data}/${sub}/ses-1/*.mb)
```

Inside each iteration of the loop, it changes to the `$pipe_dir`, removes
prior attempts at running, sets the variable `conn_proj` to be the
`${combo}/conn_make_project.mat` file, and runs

```
matlab -nodisplay -r "conn batch ${conn_proj} ; exit" 2>&1 \
    | tee ${combo}/conn-log-${sub}.txt
```

This will create the
`data/nophysio/conn/sub-737t1/ses-1/faces.mb/conn_project/results/firstlevel/SBC_01` directory that contains the files needed by `make_projec.py`.

It takes a while, as it runs `matlab` on every combination for the subjects.


Then run `make_project.py` to put data into researcher `project` directory.

## batch_make_project.sh



## conn_make_project.sh


## make_project.py

`make_project.py` is run after the `make_symlinks.sh` and
`conn_make_project.sh` scripts.

```
./bin/make_project.py \
    -s 1    # session \
    -p /nfs/turbo/lsa-lukehyde/projects/scott_amalgamated_rest_ 20220212    # project directory
    -c data/nophysio/conn    # either physio or no physio
    -d data/nophysio/preprocessed    # where the symlinks to data are
    -m specify-best    # mode, typically all or best
    -a Gordon_2016 Tian_2020_subcortical    # the atlas(es) to use
    --acq faces.mb gng.mb reward.mb    # names of data source (which task)
    --n-acqs 2    # minimum number of acquisitions to be included; default 2
    --tthresh 300    # time threshold in seconds
    --mthresh .20    # max percent of outliers from ART
    --vthresh 10    # number of coverage voxels
    --zthresh .0001    # minimum z value to be included
    --drop-key Network    # Column heading from `atlas_cluster_info.csv`
    --drop-val None    # Value from the drop-key column to not include
    --repetitionTime .8    # TR for the acquisition type, i.e., .8 for mb, 2 for sp
```

For the defaults, look in the `make_project.py` file where it parses the
arguments for the set defaults.


`drop-key Network` and `drop-val None` are needed if the Gordon atlas is needed.  
We think that `-acq` is only needed if using `-m specify-best`.
Double-check.

```
./bin/make_project.py -s 1 --n-acqs 2 \
    -p /nfs/turbo/lsa-lukehyde/projects/bennet-megarest-test \
    -c data/nophysio/conn \
    -d data/nophysio/preprocessed \
    -m best -a Gordon_2016 Tian_2020_subcortical \
    --drop-key Network \
    --drop-val None \
    --repetitionTime .8
```


The test run will then be the following.

```
./bin/make_project.py -s 1 --n-acqs 1 \
    -p /nfs/turbo/lsa-lukehyde/projects/bennet-megarest-test \
    -c data/nophysio/conn \
    -d data/nophysio/preprocessed \
    -m specify-all --acq rest.mb \
    -a Schaefer2018 Tian_2020_Subcortical \
    --repetitionTime .8
```

After that runs succesfully, we should restore the `data.hold`
directory to `data` (save the test data), and rerun both of the above
`./bin/make_project.py` commands
in their entirety.

