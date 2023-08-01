# SC23-Reproducibility
ReadMe for describing how to use Umpire's ML tool:

# How to Run

## 0) Before starting, set up your permissions

Start by emailing the Umpire team at ```umpire-dev@llnl.gov``` and cc Sam Schwartz at ```sam@cs.uoregon.edu```.
We can make sure you get set up with everything you'll need and all the permissions are granted.

## 1) Build umpire, specifically against a particular hash
```
$ git clone git@github.com:LLNL/Umpire.git

$ git checkout e6833074

$ git submodule init && git submodule update

$ mkdir build && cd build

$ cmake -DUMPIRE_ENABLE_TOOLS=On -DUMPIRE_ENABLE_BACKTRACE=On -DUMPIRE_ENABLE_BACKTRACE_SYMBOLS=Off -DCMAKE_CXX_FLAGS="-g -rdynamic" -DCMAKE_BUILD_TYPE=Debug -DUMPIRE_ENABLE_DEVELOPER_DEFAULTS=On -DCMAKE_INSTALL_PREFIX=/path/to/install ../

$ make
```
## 2) Build application against umpire, with relevant flags turned on

Example: Building against SAMARUI
```
$ lalloc 1 -W 5 cmake -DENABLE_UMPIRE=On -Dumpire_DIR=/usr/workspace/samds/2022_summer_workbench/UmpireReplay/Umpire/install/lib/cmake/umpire -DCMAKE_CXX_FLAGS="-g -rdynamic -O3" ../
```
## 3) Run application, first exporting the UMPIRE_REPLAY="On" environment variable either as a separate line:
```
$ export UMPIRE_REPLAY="On"
```
Or, continuing with our SAMARUI example (which also requires the application to be run through MPI) on the command line itself:
```
$ UMPIRE_REPLAY=On lalloc 1 -W 3 /usr/tce/packages/spectrum-mpi/ibm/spectrum-mpi-rolling-release/bin/mpiexec "-n" "1" "/usr/workspace/samds/2022_summer_workbench/SAMRAI/build_with_no_symbols_and_replay/bin/euler" "/usr/workspace/samds/2022_summer_workbench/SAMRAI/source/test/applications/Euler/test_inputs/test.2d.input"
```
As a consequence of running (3), there will be at least one output file in the same directory with the extension ```.stats```. Hold on to this/these files.

## 4) Clone tool, switch to specific commit
```
$ git clone git@github.com:samuel-schwartz/umpire-ml.git
$ git checkout bf12562
```
Ensure all needed modules are installed
```
$ cd umpire-ml
$ python3 -m pip install -r requirements.txt
```
## 5) Run tool on .stats file

```
$ python3 main.py yourfile.stats
```
The output of this will be a file called ```yourfile.stats.ml.stats```.

What happened? Big picture: Input was the raw stats file. Output was the stats file (with backtrace information removed) but with annotations added to the stats file which can be then be used in the custom replay script in (6).

## 6) Run the new .stats.ml.stats in replay

Jump back to the replay tool in the umpire installation built in (1).

Run the command

```
$ ./replay -i yourfile.stats.ml.stats -s
```

which will output information about the memory usage of the yourfile.stats as if ML had been turned on the from the get-go.

It can be compared with the initial ```yourfile.stats``` to get an idea of memory savings.


# About the ML process.

There are several steps which take place:

1. Read in file
2. An Extract, Transform, Load (ETL) pattern is applied. The extractor gets the data from the .stats/.reply files, and the data is transformed to a tabular format (e.g., CSV/TSV style data)

3. The data is split into training vs testing data, with a 80/20 split.

4. Automatic preprocssing is undertaken. This is a huge component, since we don't know what the data looks like a priori. In essence, it goes through a massive switch statement and transforms data, 1-hot encodes it, applies psuedogausian scaling, etc., based on Sam's usual "gut heuristics" when doing preprocessing.

Part of this preprocessing is encoding backtrace data in various ways (when it's included -- the software \*should\* be able to handle .stats files both with and without backtrace data in it.)


5. A mini bake-off is done across several ML models. We select model with best F1 score.

The models examined include the following SKLearn Classifiers:
classifiers = [
        LinearRegression(),
        RandomForestClassifier(),
        KNeighborsClassifier(5),
        SVC(gamma=2, C=1),
        AdaBoostClassifier(),
        GaussianNB()
    ]


6. We then create predictions on each  entire file
