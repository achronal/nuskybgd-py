# Index

- [Quick start guide](#quick-start-guide)

- [Setting up Python environment](#note-about-python-environment)


---


# Quick start guide

## Environment variables

These environment variables must be set up before anything else. Modify the
paths to point to the correct location on your machine.

```
# bash, zsh

# NUSKYBGD
export NUSKYBGD=$HOME/astro/soft/nuskybgd
export NUSKYBGD_AUXIL=$NUSKYBGD/auxil

# HEASARC CALDB
export CALDB=/soft/astro/heasarc/CALDB
export CALDBCONFIG=$CALDB/software/tools/caldb.config

# Add nuskybgd executables to $PATH
export PATH=$NUSKYBGD/bin:$PATH

# Add nuskybgd to $PYTHONPATH for its modules to be found
export PYTHONPATH=$NUSKYBGD:$PYTHONPATH

# In csh and tcsh the export statement is setenv instead (use double quotes)
# setenv PATH "$NUSKYBGD/bin:$PATH"
```

In order to not clutter your environment variables and paths, you can wrap the
above in a file and only run the export statements when you need them. For
example, you can save the above to a file `~/nuskybgd-init.sh`. Then in your
shell rc file, add an alias:

```
# bash, zsh
alias nuskybgdinit="source ~/nuskybgd-init.sh"

# csh, tcsh
# alias nuskybgdinit "source ~/nuskybgd-init.sh"
```

The next time you start a shell session, the new alias will be loaded, and you
can set the environment variables with `nuskybgdinit`.


Assuming you have the 'default' layout for files, e.g. cleaned event files are
in `[target name]/[obs id]/event_cl`, the examples below puts background
modelling related files in a folder named `bgd` inside `event_cl`.

Create the `bgd/` folder if this is the first time.

## Make an image

Use mkimgs.py to create a counts image for WCS reference.

```
# (cd into the top level)
./mkimgs.py ./ 50002031004 3 20
```

## Make instrument maps

Create image masks for the detectors. Do this for each module (`nu*A01_cl.evt`
and `nu*B01_cl.evt`) to obtain `newinstrmapA.fits` and `newinstrmapB.fits`.

```
# In event_cl/

instrmap.py nu90201039002A01_cl.evt

instrmap.py nu90201039002B01_cl.evt
```


## Aspect histogram images

Make images of the 2D histogram of the pointing position, one for each module.

```
# In event_cl/

projobs.py nu90201039002A_det1.fits gtifile=nu90201039002A01_gti.fits
out=aspecthistA.fits

projobs.py nu90201039002B_det1.fits gtifile=nu90201039002B01_gti.fits
out=aspecthistB.fits
```


## Background aperture images

Create images of the aperture background model and detector mask convolved
with the aspect. For each module, one image is created for the aperture
background and four images for the detector masks.

```
# In event_cl/bgd/

projinitbgds.py refimg=../imB3to20keV.fits out=bgdapA.fits \
	mod=A det=1234 chipmap=../newinstrmapA.fits aspect=../aspecthistA.fits

projinitbgds.py refimg=../imB3to20keV.fits out=bgdapB.fits \
	mod=B det=1234 chipmap=../newinstrmapB.fits aspect=../aspecthistB.fits
```

## Extract spectra from background regions

> **Note about regions**
>
> Region masking is handled by pyregion. If in doubt, test whether your region
> results in the mask as expected, directly with pyregion.
>
> In general:
>
> The mask is created by rendering each entry in the region file in sequence,
> changing pixels to `1` for an include region or changing pixels to `0` for
> an exclude region. Therefore, order matters! The final value in a given
> pixel depends on the last region that covers it.
>
> Is this behaviour the same in all software? I would not bet on it. It is
> particularly ambiguous if you are just looking at the regions in DS9, as to
> whether the include or exclude region takes precedence. To be completely
> safe, create regions such that all exclude regions come after all include
> regions.
>
> The same consideration should be given to region type. Stick to circle, box,
> and ellipse to be safe.

Create some background regions and extract spectra from both A and B modules,
e.g. I selected three background regions in DS9 and saved them in ds9 fk5
format, `bgd1.reg`, `bgd2.reg`, and `bgd3.reg`.

```
# New syntax for getspecnoarf.py
# In event_cl/
getspecnoarf.py nu90201039002A01_cl.evt reg=bgd/bgd1.reg \
    indir=. outdir=bgd outprefix=bgd1A \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd1A.log

getspecnoarf.py nu90201039002A01_cl.evt reg=bgd/bgd2.reg \
    indir=. outdir=bgd outprefix=bgd2A \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd2A.log

getspecnoarf.py nu90201039002A01_cl.evt reg=bgd/bgd3.reg \
    indir=. outdir=bgd outprefix=bgd3A \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd3A.log

getspecnoarf.py nu90201039002B01_cl.evt reg=bgd/bgd1.reg \
    indir=. outdir=bgd outprefix=bgd1B \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd1B.log

getspecnoarf.py nu90201039002B01_cl.evt reg=bgd/bgd2.reg \
    indir=. outdir=bgd outprefix=bgd2B \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd2B.log

getspecnoarf.py nu90201039002B01_cl.evt reg=bgd/bgd3.reg \
    indir=. outdir=bgd outprefix=bgd3B \
    attfile=../auxil/nu90201039002_att.fits.gz >& bgd/bgd3B.log
```


## Fix the spectral products' RESPFILE keywords (optional)

For spectral products from the old version of getspecnoarf.py, you can fix the
RESPFILE keyword in the PHA files.

```
# In event_cl/ or where the spectral files are
find . -iname "*.pha" -type f -exec phafix.py {} \;
```


## Create and fit the background model

Run fitab (requires PyXspec) to get `bgdparams.xcm` that has the fitted
background model within.

Create a file `bgdinfo.json` in `bgd/` with the following structure (`fitab.py
--help` will print this template). The file names for the images are default
values so you may not need to modify them, but the files for the background
regions need to be updated for your data.

```json
{
    "bgfiles": [
        "bgd1A_sr_g30.pha", "bgd1B_sr_g30.pha",
        "bgd2A_sr_g30.pha", "bgd2B_sr_g30.pha",
        "bgd3A_sr_g30.pha", "bgd3B_sr_g30.pha"
    ],

    "regfiles": [
        "bgd1A.reg", "bgd1B.reg",
        "bgd2A.reg", "bgd2B.reg",
        "bgd3A.reg", "bgd3B.reg"
    ],

    "refimgf": "bgdapA.fits",

    "bgdapfiles": {
        "A": "bgdapA.fits",
        "B": "bgdapB.fits"
    },

    "bgddetfiles": {
        "A": [
            "det0Aim.fits",
            "det1Aim.fits",
            "det2Aim.fits",
            "det3Aim.fits"
        ],
        "B": [
            "det0Bim.fits",
            "det1Bim.fits",
            "det2Bim.fits",
            "det3Bim.fits"
        ]
    }
}
```

Run fitab.py directing stdout to a log file, then check the log:

```
# In event_cl/bgd/
fitab.py bgdinfo.json savefile=IC342bgd >& fitab.log
```

Load the save file in Xspec:

```
# Start xspec, then input these commands
@IC342bgd.xcm
ignore **:**-3. 150.-**
cpd /xw
setplot energy
setplot command res y 1e-4 0.04
plot ldata delchi
```

Check how the model looks, and make any necessary adjustments. You can add
more model components, but do not remove any of the generated ones or change
their names.

When the model is OK, write the current state to a save file under a different
name to the nuskybgd generated save file. This save file contains model
parameters for the background components. The subsequent tasks will look for
these using the model component names.

```
# Creates a save file mymodel.xcm
save all mymodel
```

The saved xcm file should not contain any general XSPEC/Tcl scripts because
PyXspec will not properly execute it.


## Add detabs to rmf

Creates det0A.rmf, det1A.rmf, etc... in bgd/

```
cd /Users/qw/astro/nustar/IC342_X1/90201039002/event_cl
absrmf.py nu90201039002A01_cl.evt bgd/det
absrmf.py nu90201039002B01_cl.evt bgd/det
```



## Create fake spectra of apbgd and fcxb components

```
cd bgd/
imrefspec.py AB 0123
```


---

Extract the spectrum of an extended source in the aperture defined by src.reg.

```
# In event_cl/
mkdir spec

getspecnoarf.py nu90201039002A01_cl.evt reg=src.reg \
    indir=. outdir=spec outprefix=srcA \
    attfile=../auxil/nu90201039002_att.fits.gz >& spec/srcA.log

getspecnoarf.py nu90201039002B01_cl.evt reg=src.reg \
    indir=. outdir=spec outprefix=srcB \
    attfile=../auxil/nu90201039002_att.fits.gz >& spec/srcB.log
```

TODO

Lookup nucoord source to figure out coord transform, make instrmap use the
same method.

hard background modelling examples
ophiuchus -- cluster fills fov
A2146 -- gain shifts


---


# Note about Python environment

nuskybgd is written in Python 3. You need a Python 3 installation and PyXspec
compiled against that installation. One potential source of headache is
obtaining a Python environment in which PyXspec will work. It is recommended
that you avoid using system Python (both the executable and packages), and
instead set up your own Python installation, which you have full control over,
and won't be affected by any system update.

If you have other programs that had modified your shell rc file, to modify
your PYTHONPATH environment variable, or to change the way the Python
executable is invoked (CIAO for instance), they may be incompatible and cause
problems. If you run into problems, first check that they are not the source
of the trouble by disabling them. The set up instructions here are only
guaranteed to work in a fresh installation and vanilla environment.


## Setting up a Python 3 environment using conda

The example here uses Miniconda to install Python 3 on Ubuntu 16.04 LTS, and
sets up PyXspec using that Python installation. If you already have a Conda
installation of Python 3 but have other program dependencies layered on top,
it is best to create a fresh new environment for PyXspec and nuskybgd.

https://conda.io/projects/conda/en/latest/user-guide/install/index.html

I chose Miniconda because it just has the base conda and Python and nothing
else. If you already have conda installed, go on to create a new environment.

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

When prompted, install it into somewhere in your `$HOME` directory. By
default it is `$HOME/miniconda3`.

Run `conda init` once to modify your shell rc file (it will add itself to
`$PATH`). For example, if the shell is `zsh`,

```
$HOME/miniconda3/bin/conda init zsh
```

A snippet for conda's initialization will be inserted into `~/.zshrc`.

Start a new shell session for the modified shell rc to take effect.

Create an environment named `pyxspec` with Python 3.5

```
conda create --name pyxspec python=3.5
```

Show list of conda environments

```
conda info --envs
```

Activate:

```
conda activate pyxspec
```

Now there should be an additional `(pyxspec)` in front of the command prompt.

Verify that `python` and `pip` resolve to the new installation. The `-a`
option shows others found in your `$PATH`; the first result is the one that
will be run. If you just added a new executable to the `$PATH`, it may not be
the one selected even if it is the first entry; in that case starting a new
shell session will fix it.

```
└─ $ which -a python
/home/qw/miniconda3/envs/pyxspec/bin/python
/usr/local/bin/python
/usr/bin/python

└─ $ which -a pip
/home/qw/miniconda3/envs/pyxspec/bin/pip
/usr/local/bin/pip

└─ $ pip list
Package        Version
-------------- ---------
certifi        2018.8.24
Django         1.10.5
ipython        7.7.0
pip            19.2.2
prompt-toolkit 2.0.9
setuptools     40.2.0
wheel          0.31.1
```

A very fresh installation indeed? Not quite. In this user directory, some
packages are actually found in `.local/lib/python3.5/site-packages`, which is
where a different Python 3.5 installation placed modules when they were
installed with `pip install --user`. This is not good (see notes below about
IPython).


**Several additional checks:**

* `python3`

The nuskybgd programs are run by `#!/usr/bin/env python3`. Thus, they need to
run in an environment in which `python3` resolves to the Miniconda Python that
was just installed. Check this using `which python3`.

```
└─ $ which -a python3
/home/qw/miniconda3/envs/pyxspec/bin/python3
/usr/bin/python3
```

An additional `python3` executable is in the `$PATH`. If that one is run
instead, more likely than not there will be problems.


* `ipython`

You may want to run some nuskybgd functions in an interactive session. In that
case, you should check that `ipython` resolves to the Miniconda Python that
was just installed.

```
└─ $ which -a ipython
/usr/local/bin/ipython
```

In this case, there was no `ipython` executable in the expected location
`/home/qw/miniconda3/envs/pyxspec/bin`. A quick check with `pip` shows this:

```
└─ $ pip install ipython
Requirement already satisfied: ipython in ./.local/lib/python3.5/site-packages (7.7.0)
```

`pip` sees an existing `ipython` module in its search path and does not
install `ipython` into the Miniconda Python. This is not good because that
`ipython` module was placed there by some other Python's `pip` (installed with
the `--user` flag), may be incompatible, and can change at any time. It is
best to empty this `.local/lib/python3.5` directory altogether, and for
whatever needs the other installation have, perhaps migrate to a user managed
Python installation instead (so that there is no need to use the `--user`
flag).

After removing this folder, and installing `ipython` with `pip`,

```
└─ $ which -a ipython
/home/qw/miniconda3/envs/pyxspec/bin/ipython
/usr/local/bin/ipython
```


* `jupyter`

Install Jupyter with `pip install jupyter` and check whether the correct
executable will be run.

```
└─ $ which -a jupyter
/home/qw/miniconda3/envs/pyxspec/bin/jupyter
/usr/local/bin/jupyter
```

Although it is possible to add the Miniconda Python executable as a kernel to
another Jupyter installation, it is simplest to run the Jupyter server
installed by the Miniconda Python.


**Finally, install these packages:**

```
pip install numpy scipy astropy
```

Do this after numpy is installed:

```
pip install pyregion
```

At this point,

```
└─ $ pip list
Package            Version
------------------ ---------
astropy            3.2.1
attrs              19.1.0
backcall           0.1.0
bleach             3.1.0
certifi            2018.8.24
Cython             0.29.13
decorator          4.4.0
defusedxml         0.6.0
entrypoints        0.3
ipykernel          5.1.2
ipython            7.7.0
ipython-genutils   0.2.0
ipywidgets         7.5.1
jedi               0.15.1
Jinja2             2.10.1
jsonschema         3.0.2
jupyter            1.0.0
jupyter-client     5.3.1
jupyter-console    6.0.0
jupyter-core       4.5.0
MarkupSafe         1.1.1
mistune            0.8.4
nbconvert          5.6.0
nbformat           4.4.0
notebook           6.0.0
numpy              1.17.0
pandocfilters      1.4.2
parso              0.5.1
pexpect            4.7.0
pickleshare        0.7.5
pip                19.2.2
prometheus-client  0.7.1
prompt-toolkit     2.0.9
ptyprocess         0.6.0
Pygments           2.4.2
pyparsing          2.4.2
pyregion           2.0
pyrsistent         0.15.4
python-dateutil    2.8.0
pyzmq              18.1.0
qtconsole          4.5.3
scipy              1.3.1
Send2Trash         1.5.0
setuptools         40.2.0
six                1.12.0
terminado          0.8.2
testpath           0.4.2
tornado            6.0.3
traitlets          4.3.2
wcwidth            0.1.7
webencodings       0.5.1
wheel              0.31.1
widgetsnbextension 3.5.1
```

Install additional packages with `pip` as needed.


## Compiling PyXspec against the Miniconda Python installation

https://heasarc.nasa.gov/xanadu/xspec/python/html/buildinstall.html#changepyvers-label

Following the instructions from HEASoft, first find the correct compilation
flags. You must check that the `python-config` (or `python3-config`) actually
resolves to the Miniconda Python. On this user account, only `python3-config`
does:

```
└─ $ which -a python-config
/usr/bin/python-config

└─ $ which -a python3-config
/home/qw/miniconda3/envs/pyxspec/bin/python3-config
/usr/bin/python3-config
```

Accordingly, running `python3-config` will give the information needed. The
other one will not!

```
└─ $ python3-config --cflags
-I/home/qw/miniconda3/envs/pyxspec/include/python3.5m -I/home/qw/miniconda3/envs/pyxspec/include/python3.5m  -Wno-unused-result -Wsign-compare -march=nocona -mtune=haswell -ftree-vectorize -fPIC -fstack-protector-strong -fno-plt -O3 -pipe  -fdebug-prefix-map==/usr/local/src/conda/- -fdebug-prefix-map==/usr/local/src/conda-prefix -fuse-linker-plugin -ffat-lto-objects -flto-partition=none -flto -DNDEBUG -fwrapv -O3 -Wall -Wstrict-prototypes

└─ $ python3-config --ldflags
-L/home/qw/miniconda3/envs/pyxspec/lib/python3.5/config-3.5m -L/home/qw/miniconda3/envs/pyxspec/lib -lpython3.5m -lpthread -ldl  -lutil -lrt -lm  -Xlinker -export-dynamic
```

Modify the file `heasoft-<ver>/Xspec/BUILD_DIR/hmakerc` per instructions:

```
PYTHON_INC="-I/home/qw/miniconda3/envs/pyxspec/include/python3.5m"
PYTHON_LIB="-L/home/qw/miniconda3/envs/pyxspec/lib/python3.5/config-3.5m -L/home/qw/miniconda3/envs/pyxspec/lib -lpython3.5m"
```

Then follow the instructions to rebuild PyXspec.

**Note:**

You may not have `hmake` on your system. The HEASoft installation
itself has a `hmake` executable, which will be available after HEASoft has
been initialized, e.g. with `heainit`.

**Note 2:**

You may encounter an error in the final linking step of the build
(when `hmake` is run) with the following message about LTO:

```
lto1: fatal error: bytecode stream generated with LTO version 6.0 instead of the expected 4.1
```

The `g++` compiler on this system has a different link time optimizer version
vs. the one used to compile the Miniconda Python (against which PyXspec is
being compiled). In this case, the compiler on this Ubuntu 16.04 LTS is older.
The proper way to fix this would be to use a compiler from conda for HEASoft,
but that would be too much trouble. I copied the command for the linking step
just above the error, and appended ` -fno-lto` to the end and ran it (avoids
the issue by disabling link time optimization).


Check that PyXspec has been compiled successfully.

- In a new shell session, activate the Python environment

```
conda activate pyxspec
```

- Initialize HEASoft

```
heainit
```

- Start IPython and note the message:

```
└─ $ ipython
Python 3.5.6 |Anaconda, Inc.| (default, Aug 26 2018, 21:41:56)
Type 'copyright', 'credits' or 'license' for more information
IPython 7.7.0 -- An enhanced Interactive Python. Type '?' for help.

In [1]:
```

This is indeed the one that was just installed.

```
In [1]: import xspec

In [2]:
```

There is no error message.

Next, import `astropy.io.fits` and `pyregion`:

```
In [2]: import astropy.io.fits

In [3]: import pyregion

In [4]:
```

If they imported without problem, it is most likely that you are golden.

Finally, a note about the order of imports. I had not thought the order
mattered, but on my Mac, there is an issue with pyxpec and astropy using
different versions of libcfitsio. Since HEASoft is very up-to-date, PyXspec
uses the newer version of libcfitsio with more symbols. There, PyXspec must be
imported first to avoid the problem.



