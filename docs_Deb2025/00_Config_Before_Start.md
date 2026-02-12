# Before you start

Before running the bioinformatic scripts, the following has to be configured:

## 1. BASH_PROFILE

*This is important*

We need to set certain environmental variables. 1) `BASE_DIR` has to be set. When running a script (any script), the script will try to load this variable to find the correct location of the data. Therefore, any mention of **<YOUR_DIR>** is a self-named directory. Change **<YOUR_ID>** with your cluster ID. 2) Set the root directory for every virtual environment with micromamba. 

So, do the following:

1. Open the bash_profile: `vim ~/.bash_profile`

2. Add these lines 

```
# Colors
export LC_CTYPE="en_US.UTF-8"
export LC_ALL="en_US.UTF-8"
export PS1='\u@\H:\w$ ';

# BASE_DIR
export BASE_DIR="/omics/groups/OE0532/internal/YOUR_DIR/"

# for .libPaths()
export R4_2_DIR="/home/<YOUR_ID>/R/x86_64-pc-linux-gnu-library/4.2"
export R4_DIR="/omics/groups/OE0532/internal/RStudio"
export R3_6_DIR="/omics/groups/OE0532/internal/RStudio_3.6.2"
export R3_4_DIR="/omics/groups/OE0532/internal/RStudio_3.4"

export PATH=${JAVA_HOME}/bin:${PATH}

# micromamba
export MAMBA_ROOT_PREFIX="/home/YOUR_ID/Micromamba/"
# module load Micromamba/2.0.2-0               ### Uncomment and remove this comment after installing p3!
eval "$(micromamba shell hook --shell bash)"
# micromamba activate p3                     ### Uncomment and remove this comment after installing p3!


#RStudio
if ps -e | grep -q rserver; then
        module load gcc/7.2.0
        module load libpng/1.6.37
        module load hdf5/1.8.18

fi
```


Quit bash profile. Remember to name <YOUR_DIR> in the code above!

3. Create this directory: `/omics/groups/OE0532/internal/YOUR_DIR`

4. Create a symlink to access static data: `ln -s /omics/groups/OE0532/internal/from_snapshot/static /omics/groups/OE0532/internal/YOUR_DIR/`

5. Create a symlink for the software dir: `ln -s  /omics/groups/OE0532/internal/from_snapshot/software /omics/groups/OE0532/internal/YOUR_DIR/`

6. Reload bash profile: `source ~/.bash_profile`


### RStudio

Different R scripts require different R versions. For each version there is a location to install packages. 

This can be changed with the function `.libPaths()` (this is already included in each script). Thus, the following lines are already included in your bash_profile. 

```
export R4_DIR="/omics/groups/OE0532/internal/RStudio"
export R3_6_DIR="/omics/groups/OE0532/internal/RStudio_3.6.2"
export R3_4_DIR="/omics/groups/OE0532/internal/RStudio_3.4"
```


## 2. Virtual env for python3

Most scripts will be run under python 3.

1. Load micromamba: `module load Micromamba/2.0.2-0`

2. Initialization of micromamba **(Needs to be done only once, Skip if done before)**
```
micromamba shell init --shell bash --root-prefix=~/Micromamba
```

3. Create your python 3 environment with important packages/ dependencies:
```
micromamba create --name p3 --file /omics/groups/OE0532/internal/Andres/Deb_Config_files/p3.yml
```

4. Once installation is done, activate env: `micromamba activate p3`

5. Advice: Now add `module load Micromamba/2.0.2-0` and `micromamba activate p3` to your bash profile, to automatically load the package and activate this virtual env.



## 3. Virtual env for python2

Python2 is mainly used by diricore and couple of more tools.

1. Load micromamba: `module load Micromamba/2.0.2-0`

2. Initialization of micromamba **(Needs to be done only once, Skip if done before)**
```
micromamba shell init --shell bash --root-prefix=~/Micromamba
```

3. Create your python 2 environment with important packages/ dependencies:
```
micromamba create --name diricore --file /omics/groups/OE0532/internal/Andres/Deb_Config_Files/diricore.yml

```

4. Once installation is done, activate env: `micromamba activate diricore`

5. Install htseq package separately. Change **<YOUR_ID>** with your cluster ID:
```
/home/YOUR_ID/Micromamba/envs/diricore/bin/python -m pip install htseq==0.11.3
```

6. Install RiboDiff separately. If **not** downloaded yet: `git clone git@github.com:kate-v-stepanova/RiboDiff.git`.
Next steps need to be done to install RiboDiff in your new virtual env.
```
cd RiboDiff
python setup.py build
python setup.py install
```

## After installing both environments, restart bash profile by: 
```
source ~/.bash_profile
```






