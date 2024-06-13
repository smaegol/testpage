# Working with nanopore file formats
Agnieszka Czarnocka-Cieciura,Pawel S Krawczyk,Natalia Gumińska

- [Working with nanopore file formats](#working-with-nanopore-file-formats)
- [Prerequisities](#prerequisities)
  - [Software](#software)
  - [Data](#data)
- [Sequencing output](#sequencing-output)
  - [Folder name](#folder-name)
  - [Folder structure](#folder-structure)
- [POD5 files](#pod5-files)
  - [introduction](#introduction)
  - [pod5 python library](#pod5-python-library)
    - [Viewing pod5 files.](#viewing-pod5-files)
    - [Read details](#read-details)
  - [Format conversion](#format-conversion)
      - [POD5 to FAST5](#pod5-to-fast5)
      - [FAST5 to POD5](#fast5-to-pod5)
- [FAST5 files](#fast5-files)
  - [Inspecting fast5 files](#inspecting-fast5-files)

# Prerequisities

## Software

It’s assumed that all steps of analysis are run in the Linux
environment. Installation instructions can be found on the documentation
websites for each of the program.

Required software packages:

- pod5 (<https://github.com/nanoporetech/pod5-file-format>)
- ont-fast5-api (<https://github.com/nanoporetech/ont_fast5_api>)

If they are not installed yet, use below commands:

    pip install pod5
    pip install ont_fast5_api

## Data

For the purpose of exercises data located in /home/kurs/raw will be
used.

# Sequencing output

## Folder name

Data from the single sequencing run are stored in the folders named in a
format DATE_TIME_DEVID_FLOWCELLID_RUNID (for example:
20240605_1147_MN32794_FAX66946_9e2c432c), where:

- **DATE** - date (YYYYMMDD) when the sequencing started
- **TIME** - time (HHMM) when the sequencing started
- **DEVID** - unique ID of the device used for sequencing (e.g. MN32794
  for MinION 32794)
- **FLOWCELLID** - unique ID of the flowcell used for sequencing
  (e.g. FAX66946)
- **RUNID** - unique identifier of the current run


> [!CAUTION]
> Advises about risks or negative outcomes of certain actions.


> [!NOTE]
> **Exercises:**
>
> 1.  Please identify of the elements of folder names in the folder
>     /home/kurs/raw
> 2.  Please list the content of each folder



## Folder structure

As you can see from the output of the last exercise, there are multiple
folders and files in the folder containing raw sequencing output. Some
of them we will use later for sequencing QC.

| folder                 | description                                                                                                                                                            |
|------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| pod5                   | raw pod5 files (will be used for basecalling)                                                                                                                          |
| report\_\*             | files containing the report summarising the sequencing run. HTML file can be opened in the web browser                                                                 |
| sample_sheet\_\*       | list of samples in the sequencing run (contains multiple entries in case of barcoded libraries)                                                                        |
| throughput\_\*         | table summarising the sequencing output (reads,bases, etc) throughout the duration of the sequencing                                                                   |
| pore_activity\_\*      | table summarising the health of the flowcell during the sequencing                                                                                                     |
| final_summary\_\*      | File containing information on the instrument and flowcell IDs, sequencing protocol used, number of files produced and time to sequencing                              |
| other_reports          | Folder with the report on pore activity scanning                                                                                                                       |
| fastq_pass, fastq_fail | If live basecalling was enabled these folders will contain basecalled fastq files                                                                                      |
| bam_pass, bam_fail     | If reference mapping was enabled during sequencing these folders will contain bam files with results of mapping                                                        |
| sequencing_summary\_\* | table with basic information on each sequenced read, including: read_id, names of files containing this read (pod5, fastq, fast5, bam), read length, mean quality, etc |

<div>

> **Exercises**
>
> 1.  Check the content of selected files using command line tools (cat,
>     more/less).
> 2.  Find as much as possible about read with id
>     “ea0389c8-b3e1-475f-976a-ef2b676a21ac”

</div>

<div>

> **Tip**
>
> use `less -S` to prevent line wrapping when viewing wide files.

</div>

# POD5 files

## introduction

POD5 is a file format for storing nanopore data in an easily accessible
way. The format is able to be written in a streaming manner which allows
a sequencing instrument to directly write the format.

Data in POD5 is stored using Apache Arrow, allowing users to consume
data in many languages using standard tools.

The information about the file format is available at [nanoporetech
github](https://github.com/nanoporetech/pod5-file-format).

## pod5 python library

Direct work with pod5 files requires the
([pod5](https://pypi.org/project/pod5/)) python library. Together with
the library a basic toolkit for inspecting, converting, subsetting and
formatting POD5 files is installed.

``` bash
usage: pod5 [-h] [-v] {convert,inspect,merge,repack,subset,filter,recover,update,view} ...

**********      POD5 Tools      **********

Tools for inspecting, converting, subsetting and formatting POD5 files

optional arguments:
  -h, --help            show this help message and exit
  -v, --version         Show pod5 version and exit.

Example: pod5 convert fast5 input.fast5 --output output.pod5
```

### Viewing pod5 files.

To view the content of pod5 files the `view` subprogram can be used.

``` bash
pod5 view file.pod5
```

It will return a tab separated table containing following columns (based
on `pod5 view -L`):

- **read_id** Read UUID
- **filename** Source pod5 filename
- **read_number** Read number
- **channel** 1-indexed channel
- **mux** 1-indexed well
- **end_reason** End reason string
- **start_time** Seconds since the run start to the first sample of this
  read
- **start_sample** Samples recorded on this channel since run start to
  the first sample of this read
- **duration** Seconds of sampling for this read
- **num_samples** Number of signal samples
- **minknow_events** Number of minknow events that this read contains
- **sample_rate** Number of samples recorded each second
- **median_before** Current level in this well before the read
- **predicted_scaling_scale** Scale for predicted read scaling
- **predicted_scaling_shift** Shift for predicted read scaling
- **tracked_scaling_scale** Scale for tracked read scaling
- **tracked_scaling_shift** Shift for tracked read scaling
- **num_reads_since_mux_change** Number of selected reads since the last
  mux change on this channel
- **time_since_mux_change** Seconds since the last mux change on this
  channel
- **run_id** Run UUID
- **sample_id** User-supplied name for the sample
- **experiment_id** User-supplied name for the experiment
- **flow_cell_id** The flow cell id
- **pore_type** Name of the pore in this well

<div>

> **Exercises**
>
> 1.  Navigate to the `pod5` subfolder in the raw data folder and
>     inspect few of the files present in the folder.
> 2.  How many rows (reads) has each file?
> 3.  For selected pod5 file, output all read ids to file.

</div>

### Read details

It is also possible to get more detailed information on each read using
`pod5 inspect read` command.

<div>

> **Exercises**
>
> 1.  Get detailed information on first read in selected pod5 file.
> 2.  What was the sequencing kit used for sequencing?
> 3.  What was the version of MinKNOW software used?

</div>

## Format conversion

The most up-to-date software packages provided by the ONT (like dorado
basecaller) are optimized for work with pod5 files. However, for a long
time fast5 format was in use, so there is a lot of publicly available
datasets with data in fast5 format, as well as software packages working
on fast5 files. Thus, it is sometimes required to perform format
conversion. POD5 library can be used to convert both from pod5 to fast5
as well as from fast5 to pod5.

#### POD5 to FAST5

To convert all the pod5 files in the pod5 folder to fast5 files in the
fast5 folder, please use the command:

    pod5 convert to_fast5 -o fast5/ pod5/*.pod5

#### FAST5 to POD5

Conversion in the opposite direction (fast5 -\> pod5) is possible in the
similar way. However, it is required to decide if all fast5 are
converted into single pod5 file, or each fast5 will be converted to
separate pod5 (resembling original structure).

All the options are listed below:

    kurs@nanopore-kurs:~$ pod5 convert from_fast5 --help
    usage: pod5 convert fast5 [-h] -o OUTPUT [-r] [-t THREADS] [--strict] [-O ONE_TO_ONE] [-f] [--signal-chunk-size SIGNAL_CHUNK_SIZE] inputs [inputs ...]

    Convert fast5 file(s) into a pod5 file(s)

    positional arguments:
      inputs                Input path for fast5 file

    optional arguments:
      -h, --help            show this help message and exit
      -r, --recursive       Search for input files recursively matching `*.pod5` (default: False)
      -t THREADS, --threads THREADS
                            Set the number of threads to use (default: 4)
      --strict              Immediately quit if an exception is encountered during conversion instead of continuing with remaining inputs after issuing a warning (default: False)

    required arguments:
      -o OUTPUT, --output OUTPUT
                            Output path for the pod5 file(s). This can be an existing directory (creating 'output.pod5' within it) or a new named file path. A directory must be given when using --one-to-one.
                            (default: None)

    output control arguments:
      -O ONE_TO_ONE, --one-to-one ONE_TO_ONE
                            Output files are written 1:1 to inputs. 1:1 output files are written to the output directory in a new directory structure relative to the directory path provided to this argument. This
                            directory path must be a relative parent of all inputs. (default: None)
      -f, --force-overwrite
                            Overwrite destination files (default: False)
      --signal-chunk-size SIGNAL_CHUNK_SIZE
                            Chunk size to use for signal data set (default: 102400)

To convert the whole run with fast5 files, use a command:

    pod5 convert from_fast5 <folder_with_fast5> -r  --output ./pod5 --one-to-one ./ -t 30

This will convert each fast5 file to pod5 file in a 1:1 manner. Output
files will be saved in the `pod5` folder.

<div>

> **Exercises**
>
> 1.  Convert pod5 files located in the sample1 folder to fast5
> 2.  Convert back from fast5 to pod5

</div>

# FAST5 files

Fast5 for a long time was a major data format used by ONT. It is based
on the [HDF5 format](https://www.hdfgroup.org/solutions/hdf5/). There is
multiple software packages that can be used for viewing and manipulation
of HDF5 files, including python and R libraries:

- [libhdf5 linux library](https://www.hdfgroup.org/downloads/hdf5)
- [h5py](https://docs.h5py.org/en/stable/index.html) - pythonic
  interface to the HDF5 binary data format.
- [rhdf5](https://bioconductor.org/packages/release/bioc/html/rhdf5.html) -
  R library - an interface between HDF5 and R
- [ont_fast5_api](https://github.com/nanoporetech/ont_fast5_api) -
  simple Python interface to HDF5 files of the Oxford Nanopore .fast5
  file format.

## Inspecting fast5 files

To show the structure of fast5 file, h5ls and h5dump commands can be
used:
