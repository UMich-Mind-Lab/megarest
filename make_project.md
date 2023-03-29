## make_project.py

`make_project.py` is run after the `make_symlinks.sh` and `conn_make_project.sh` scripts.

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
    
    