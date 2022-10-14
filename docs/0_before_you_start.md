# Before you start

Before running the bioinformatic scripts, the following has to be configured:

## 1. BASE_DIR

!! This is important!

`BASE_DIR` has to be set as an environmental variable. When running a script (any script), the script will try to load this variable to find the correct location of the data. Therefore, any mention of `YOUR_DIR` is a self-named directory. 

So, do the following:

1. Open the bash_profile: `vim ~/.bash_profile`

2. And add this line: `export BASE_DIR="/omics/groups/OE0532/internal/YOUR_DIR"` 

(and also run it in your terminal - if you want to use BASE_DIR immediately)

Quit bash profile.

3. Create a directory: `/omics/groups/OE0532/internal/YOUR_DIR`

4. Create a symlink to access static data: `ln -s /omics/groups/OE0532/internal/from_snapshot/static /omics/groups/OE0532/internal/YOUR_DIR/`

5. Create a symlink for the software dir: `ln -s  /omics/groups/OE0532/internal/from_snapshot/software /omics/groups/OE0532/internal/YOUR_DIR/`

## 2. RStudio

Different R scripts require different R versions. For each version there is a location to install packages. 

This can be changed with the function `.libPaths()` (this is already included in each script)

The following variables has to be set in `.bash_profile`: `vim ~/.bash_profile`

```
export R4_DIR="/omics/groups/OE0532/internal/RStudio"
export R3_6_DIR="/omics/groups/OE0532/internal/RStudio_3.6.2"
export R3_4_DIR="/omics/groups/OE0532/internal/RStudio_3.4"
```

## 3. Configure .bash_profile (optionally)

For more convenient work, `~/.bash_profile` can be configured as following: 

```
# colors
export LC_CTYPE="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
export LC_CTYPE="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
export PS1='\u@\H:\w$ ';
export PS1="\[${cyan}\](\$(basename \$CONDA_DEFAULT_ENV 2> /dev/null)) "$PS1;

# BASE_DIR
export BASE_DIR="/omics/groups/OE0532/internal/YOUR_DIR/"

# for .libPaths()
export R4_DIR="/omics/groups/OE0532/internal/RStudio"
export R3_6_DIR="/omics/groups/OE0532/internal/RStudio_3.6.2"
export R3_4_DIR="/omics/groups/OE0532/internal/RStudio_3.4"


# conda
module load anaconda2/2019.07

# RStudio
if ps -e | grep -q rserver; then
  module load gcc/7.2.0
  module load libpng/1.6.37
  module load hdf5/1.8.18
  cd /icgc/dkfzlsdf/analysis/OE0532/tmp/RStudio_data
else
    source activate p3
fi
```

## 4. Virtual env for python3

Mainly the scripts will be running under python 3.

1. Load anaconda: `module load anaconda2/2019.07`

2. Create conda environment: `conda create -n p3 python=3.7` 

3. Activate env: `conda activate p3` or `source activate p3`

4. Install dependencies: `pip install -r /omics/groups/OE0532/internal/from_snapshot/p3_packages.txt`


## 5. Virtual env for python2

Python2 is mainly used by diricore and couple of more tools.

1. Load anaconda:  `module load anaconda2/2019.07`

2. Create conda environment: `conda create -n diricore python=2.7` 

3. Activate env: `conda activate diricore` or `source activate diricore`

3. Install dependencies: `pip install -r /omics/groups/OE0532/internal/Alex/diricore_packages.txt`

After installing both environments, restart bash profile by: 
```
cd
source ~/.bash_profile
```






