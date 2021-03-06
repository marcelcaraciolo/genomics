Illumina data handling utilities
================================

Utilities for preparing data on the cluster from the Illumina instrument:

* :ref:`analyse_illumina_run`: reporting and manipulations of Illumina run data
* :ref:`auto_process_illumina`: automatically process Illumina-based sequencing run
* :ref:`bclToFastq`: generate FASTQ from BCL files
* :ref:`build_illumina_analysis_dirs`: create and populate per-project analysis dirs
* :ref:`demultiplex_undetermined_fastq`: demultiplex undetermined Illumina reads
* :ref:`prep_sample_sheet`: edit SampleSheet.csv before generating FASTQ
* :ref:`report_barcodes`: analyse barcode sequences from FASTQ files
* :ref:`rsync_seq_data`: copy sequencing data using rsync
* :ref:`verify_paired`: utility to check FASTQs form R1/R2 pair

.. _analyse_illumina_run:

analyse_illumina_run.py
***********************

Utility for performing various checks and operations on Illumina data.

Usage::

    analyse_illumina_run.py OPTIONS illumina_data_dir

``illumina_data_dir`` is the top-level directory containing the ``Unaligned`` directory
with the fastq.gz files produced by the BCL-to-FASTQ conversion step.

Options:

.. cmdoption:: --report

    report sample names and number of samples for each project

.. cmdoption:: --summary

    short report of samples (suitable for logging file)

.. cmdoption:: -l, --list

    list projects, samples and fastq files directories

.. cmdoption:: --unaligned=UNALIGNED_DIR

    specify an alternative name for the 'Unaligned' directory conatining the
    fastq.gz files

.. cmdoption:: --copy=COPY_PATTERN

    copy fastq.gz files matching COPY_PATTERN to current directory

.. cmdoption:: --verify=SAMPLE_SHEET

    check CASAVA outputs against those expected for ``SAMPLE_SHEET``

.. cmdoption:: --stats

    Report statistics (read counts etc) for fastq files

.. _auto_process_illumina:

auto_process_illumina.sh
************************

Automatically process data from an Illumina-based sequencing platform

Usage::

    auto_process_illumina.sh COMMAND [ PLATFORM DATA_DIR ]

``COMMAND`` can be one of::

    setup: prepares a new analysis directory. This step must be
           done first and requires that PLATFORM and DATA_DIR 
           arguments are also supplied (these do not have to be
           specified for other commands).
           This creates an analysis directory in the current dir
           with a custom_SampleSheet.csv file; this should be
           examined and edited before running the subsequent 
           steps.

    make_fastqs: runs CASAVA to generate Fastq files from the
           raw bcls.

    run_qc: runs the QC pipeline and generates reports.

The make_fastqs and run_qc commands must be executed from the
analysis directory created by the setup command.

Standard protocol
-----------------

The ``auto_process_illumina.sh`` script is intended to automate the major
steps in generating FASTQ files from raw Illumina BCL data.

The standard protocol for using the automated script is:

1. Run the ``setup`` step to create a new analysis directory
2. Move into the analysis directory
3. **Check and if necessary edit the generated sample sheet, based on
   the predicted output projects and samples**
4. **Check and if necessary edit the bases mask setting in the
   ``DEFINE_RUN`` line in the ``processing.info`` file**
5. Run the ``make_fastqs`` step
6. Inspect the summary file which lists the generated FASTQ files
   along with their sizes and number of reads (and number of
   undetermined reads)
7. Run the ``run_qc`` step

The critical step is to check and edit the sample sheet, as this is used
to determine which samples are assigned to which project. After editing
the sample sheet it is a good idea to check the predicted outputs by
running::

    prep_sample_sheet.py SAMPLE_SHEET

and ensure that this is what was actually intended, before running the
next steps.

To change the settings used by CASAVA's BCL to FASTQ conversion, it is
also necessary to edit the ``DEFINE_RUN`` line in the ``processing.info``
file. This line typically looks like::

    DEFINE_RUN	custom_SampleSheet.csv:Unaligned:y68,I7

The colon-delimited values are:

* Sample sheet name in the analysis directory (default:
  ``custom_SampleSheet.csv``)
* The output directory where CASAVA will write the output data file
  (default: ``Unaligned``)
* The bases mask that will be used by CASAVA (default will be
  determined automatically from the ``RunInfo.xml`` file in the
  source data directory)

Optionally a fourth colon-delimited value can be supplied:

* The number of allowed mismatches when demultiplexing (default will
  be determined from the bases mask value)

Multiple samplesheets
---------------------

In some cases it might be necessary to split the BCL to FASTQ processing
across multiple sample sheets.

In this case the protocol would be:

1. Run the ``setup`` step
2. Move into the analysis directory
3. **Create multiple sample sheets as required**
4. **Edit the `processing.info` file to add `DEFINE_RUN` for each
   sample sheet**
5. Run the ``make_fastqs`` step, which will automatically run a separate
   BCL to FASTQ conversion for each ``DEFINE_RUN`` line
6. For each BCL to FASTQ conversion, inspect the summary file which
   lists the generated FASTQ files along with their sizes and number of
   reads (and number of undetermined reads)
7. Run the ``run_qc`` step, which will automatically run a separate QC on
   the outputs of each BCL to FASTQ conversion

The previous section has  more detail on the format and content of the
``DEFINE_RUN`` line. In the case of multiple ``DEFINE_RUN`` lines, it is
advised to specify distinct output directories, e.g.::

    DEFINE_RUN	pjbriggs_SampleSheet.csv:Unaligned_pjbriggs:y68,I7

.. _bclToFastq:

bclToFastq.sh
*************

Bcl to Fastq conversion wrapper script

Usage::

    bclToFastq.sh <illumina_run_dir> <output_dir>

``<illumina_run_dir>`` is the top-level Illumina data directory; Bcl files are expected to
be in the ``Data/Intensities/BaseCalls`` subdirectory. ``<output_dir>`` is the top-level
target directory for the output from the conversion process (including the generated fastq
files).

The script runs ``configureBclToFastq.pl`` from ``CASAVA`` to set up conversion scripts,
then runs ``make`` to perform the actual conversion. It requires that ``CASAVA`` is
available on the system.

Options:

.. cmdoption:: --nmismatches N

   set number of mismatches to allow; recommended values are 0 for
   samples without multiplexing, 1 for multiplexed samples with tags
   of length 6 or longer (see the CASAVA user guide for details of
   the ``--nmismatches`` option)

.. cmdoption:: --use-bases-mask BASES_MASK

   specify a bases-mask string tell CASAVA how to use each cycle.
   The supplied value is passed directly to configureBcltoFastq.pl
   (see the CASAVA user guide for details of how --use-bases-mask
   works)

.. cmdoption:: --nprocessors N

   set the number of processors to use (defaults to 1).
   This is passed to the -j option of the 'make' step after running
   configureBcltoFastq.pl (see the CASAVA user guide for details of
   how -j works)

.. _build_illumina_analysis_dirs:

build_illumina_analysis_dirs.py
*******************************

Query/build per-project analysis directories for post-bcl-to-fastq data from Illumina GA2
sequencer.

Usage::

    build_illumina_analysis_dir.py OPTIONS illumina_data_dir

Create per-project analysis directories for Illumina run. ``illumina_data_dir``
is the top-level directory containing the ``Unaligned`` directory with the
fastq.gz files generated from the bcl files. For each ``Project_...`` directory
build_illumina_analysis_dir.py makes a new subdirectory and populates with
links to the fastq.gz files for each sample under that project.

Options:

.. cmdoption:: --dry-run

    report operations that would be performed if creating the analysis directories
    but don't actually do them

.. cmdoption:: --unaligned=UNALIGNED_DIR

    specify an alternative name for the ``Unaligned`` directory conatining the fastq.gz
    files

.. cmdoption:: --expt=EXPT_TYPE

    specify experiment type (e.g. ChIP-seq) to append to the project name when creating
    analysis directories. The syntax for ``EXPT_TYPE`` is ``<project>:<type>`` e.g.
    ``--expt=NY:ChIP-seq`` will create directory ``NY_ChIP-seq``. Use multiple
    ``--expt=...`` to set the types for different projects

.. cmdoption:: --keep-names

    preserve the full names of the source fastq files when creating links

.. cmdoption:: --merge-replicates

    create merged fastq files for each set of replicates detected

.. _demultiplex_undetermined_fastq:

demultiplex_undetermined_fastq.py
*********************************

Demultiplex undetermined Illumina reads output from CASAVA.

Usage::

    demultiplex_undetermined_fastq.py OPTIONS DIR

Reassign reads with undetermined index sequences. (i.e. barcodes). DIR is the
name (including any leading path) of the 'Undetermined_indices' directory
produced by CASAVA, which contains the FASTQ files with the undetermined reads
from each lane.

Options:

.. cmdoption:: --barcode=BARCODE_INFO

    specify barcode sequence and corresponding sample name as ``BARCODE_INFO``.
    The syntax is ``<name>:<barcode>:<lane>`` e.g. ``--barcode=PB1:ATTAGA:3``

.. cmdoption:: --samplesheet=SAMPLE_SHEET

    specify SampleSheet.csv file to read barcodes, sample names and lane
    assignments from (as an alternative to ``--barcode``).

.. _prep_sample_sheet:

prep_sample_sheet.py
********************

Prepare sample sheet files for Illumina sequencers for input into CASAVA.

Usage::

    prep_sample_sheet.py [OPTIONS] SampleSheet.csv

Utility to prepare SampleSheet files from Illumina sequencers. Can be used to
view, validate and update or fix information such as sample IDs and project
names before running BCL to FASTQ conversion.

Options:

.. cmdoption:: -o SAMPLESHEET_OUT

    output new sample sheet to ``SAMPLESHEET_OUT``

.. cmdoption:: -v, --view

    view contents of sample sheet

.. cmdoption:: --fix-spaces

    replace spaces in SampleID and SampleProject fields with underscores

.. cmdoption:: --fix-duplicates

    append unique indices to SampleIDs where original SampleID/SampleProject
    combination are duplicated

.. cmdoption:: --fix-empty-projects

    create SampleProject names where these are blank in the original sample sheet

.. cmdoption:: --set-id=SAMPLE_ID

    update/set the values in the 'SampleID' field;
    SAMPLE_ID should be of the form ``<lanes>:<name>``,
    where ``<lanes>`` is a single integer (e.g. 1), a set of
    integers (e.g. 1,3,...), a range (e.g. 1-3), or a
    combination (e.g. 1,3-5,7)

.. cmdoption:: --set-project=SAMPLE_PROJECT

    update/set values in the 'SampleProject' field;
    ``SAMPLE_PROJECT`` should be of the form '<lanes>:<name>',
    where <lanes> is a single integer (e.g. 1), a set of
    integers (e.g. 1,3,...), a range (e.g. 1-3), or a
    combination (e.g. 1,3-5,7)

.. cmdoption:: --ignore-warnings

    ignore warnings about spaces and duplicated sampleID/sampleProject
    combinations when writing new samplesheet.csv file

.. cmdoption:: --include-lanes=LANES

    specify a subset of lanes to include in the output sample sheet;
    ``LANES`` should be single integer (e.g. 1), a list of integers (e.g.
    1,3,...), a range (e.g. 1-3) or a combination (e.g. 1,3-5,7).
    Default is to include all lanes

.. cmdoption:: --truncate-barcodes=BARCODE_LEN

    trim barcode sequences in sample sheet to number of bases specified
    by ``BARCODE_LEN``. Default is to leave barcode sequences unmodified

Deprecated options:

.. cmdoption:: --miseq

    convert MiSEQ input sample sheet to CASAVA-compatible format (deprecated;
    conversion is performed automatically if required)


Examples:

1. Read in the sample sheet file ``SampleSheet.csv``, update the ``SampleProject``
   and ``SampleID`` for lanes 1 and 8, and write the updated sample sheet to the
   file ``SampleSheet2.csv``::

     prep_sample_sheet.py -o SampleSheet2.csv --set-project=1,8:Control \
          --set-id=1:PhiX_10pM --set-id=8:PhiX_12pM SampleSheet.csv

2. Automatically fix spaces and duplicated ``sampleID``/``sampleProject``
   combinations and write out to ``SampleSheet3.csv``::

     prep_sample_sheet.py --fix-spaces --fix-duplicates \
          -o SampleSheet3.csv SampleSheet.csv

.. _report_barcodes:

report_barcodes.py
******************

Examine barcode sequences from one or more Fastq files and report the most
prevalent. Sequences will be pooled from all specified Fastqs before being
analysed.

Usage::

    report_barcodes.py FASTQ [FASTQ...]

Options:

.. cmdoption:: --cutoff=CUTOFF

    Minimum number of times a barcode sequence must appear to
    be reported (default is 1000000)

.. _rsync_seq_data:

rsync_seq_data.py
*****************

Rsync sequencing data to archive location, inserting the correct 'year' and
'platform' subdirectories.

Usage::

    rsync_seq_data.py [OPTIONS] DIR BASE_DIR

Wrapper to rsync sequencing data: DIR will be rsync'ed to a subdirectory of
BASE_DIR constructed from the year and platform i.e. BASE_DIR/YEAR/PLATFORM/.
YEAR will be the current year (over-ride using the --year option), PLATFORM
will be inferred from the DIR name (over-ride using the --platform option).
The output from rsync is written to a file rsync.DIR.log.

Options:

.. cmdoption:: --platform=PLATFORM

    explicitly specify the sequencer type

.. cmdoption:: --year=YEAR

    explicitly specify the year (otherwise current year is assumed)

.. cmdoption:: --dry-run

    run rsync with ``--dry-run`` option

.. cmdoption:: --chmod=CHMOD

    change file permissions using ``--chmod`` option of rsync (e.g
    'u-w,g-w,o-w')

.. cmdoption:: --exclude=EXCLUDE_PATTERN

    specify a pattern which will exclude any matching
    files or directories from the rsync

.. cmdoption:: --mirror

    mirror the source directory at the destination (update
    files that have changed and remove any that have been
    deleted i.e. rsync --delete-after)

.. cmdoption:: --no-log

    write rsync output directly stdout, don't create a log file

.. _verify_paired:

verify_paired.py
****************

Utility to verify that two fastq files form an R1/R2 pair.

Usage::

    verify_paired.py OPTIONS R1.fastq R2.fastq

Check that read headers for R1 and R2 fastq files are in agreement, and that
the files form an R1/2 pair.
