================
Data Preparation
================

Displaying large amounts of data often requires first turning it into
not-so-large amounts of data. Clodius is a program and library designed
to aggregate large datasets to make them easy to display at different
resolutions.

Bed Files
---------

BED files specify genomic intervals. They are aggregated according to an
importance function that determines which values should be visible at lower
zoom levels. This importance function is user specified. In the absence of
any clear ranking of the different lines in the BED file, a random value
can be used in lieu of the importance function.

Example BED file:

.. code-block:: bash

    chr9    135766734       135820020       TSC1    Biallelic inactivation may predict sensitivity to MTOR inhibitors
    chr16   2097895 2138721 TSC2    Biallelic inactivation may predict sensitivity to MTOR inhibitors
    chr3    10183318        10195354        VHL     May signal the presence of a germline mutation.
    chr11   32409321        32457081        WT1     May signal the presence of a germline mutation.

This file can be aggregated like so:

.. code-block:: bash

    clodius aggregate bedfile \
        --assembly hg19 \
        short.bed

And then imported into higlass after copying to the docker temp directory (``cp short.bed.multires ~/hg-tmp/``):

.. code-block:: blash

     docker exec higlass-container python \
        higlass-server/manage.py ingest_tileset \
            --filename /tmp/short.bed.multires \
            --filetype beddb \
            --datatype bedlike \
            --coordSystem b37

Bedpe-like Files
----------------

BEDPE-like files contain two sets of chromosomal coordinates:

.. code-block:: bash
    
    chr10   74160000        74720000    chr10   74165000    74725000
    chr12   120920000       121640000   chr12   120925000   121645000
    chr15   86360000        88840000    chr15   86365000    88845000

To view such files in HiGlass, they have to be aggregated so that tiles don't
contain too many values and slow down the renderer:

.. code-block:: bash

    clodius aggregate bedpe \
        --assembly hg19 \
        --chr1-col 1 --from1-col 2 --to1-col 3 \
        --chr2-col 4 --from2-col 5 --to2-col 6 \
        --output-file domains.txt.multires \
        domains.txt

This requires the `--chr1-col`, `--from1-col`, `--to1-col`, `--chr2-col`,
`--from2-col`, `--to2-col` parameters to specify which columns in the datafile
describe the x-extent and y-extent of the region.

The priority with which regions are included in lower resolution tiles is
specified by the `--impotance-column` parameter. This can either provide a
value, contain `random`, or if it's not specified, default to the size of the
region.

Epilogos Data
-------------

Epilogos (https://epilogos.altiusinstitute.org/) show the distribution of chromatin states
over a set of experimental conditions (e.g. cell lines). The data consist of positions and
states::

    chr1    10000   10200   id:1,qcat:[ [-0.2833,15], [-0.04748,5], [-0.008465,7], [0,2], [0,3], [0,4], [0,6], [0,10], [0,11], [0,12], [0,13], [0,14], [0.0006647,1], [0.436,8], [1.921,9] ]
    chr1    10200   10400   id:2,qcat:[ [-0.2833,15], [-0.04748,5], [0,3], [0,4], [0,6], [0,7], [0,10], [0,11], [0,12], [0,13], [0,14], [0.0006647,1], [0.004089,2], [0.8141,8], [1.706,9] ]
    chr1    10400   10600   id:3,qcat:[ [-0.2588,15], [-0.04063,5], [0,2], [0,3], [0,4], [0,6], [0,7], [0,10], [0,11], [0,12], [0,13], [0,14], [0.0006647,1], [0.2881,8], [1.58,9] ]
    chr1    10600   10800   id:4,qcat:[ [-0.02619,15], [0,1], [0,2], [0,3], [0,4], [0,6], [0,7], [0,8], [0,10], [0,11], [0,12], [0,13], [0,14], [0.1077,5], [0.4857,9] ]

This can be aggregated into multivec format:

.. code-block:: bash

    clodius convert bedfile_to_multivec \
        hg38/all.KL.bed.gz \
        --assembly hg38 \
        --starting-resolution 200 \
        --row-infos-filename row_infos.txt \
        --num-rows 15 \
        --format epilogos

BedGraph files
--------------

.. warning:: The order of the chromosomes in the bedgraph file have to
    be consistent with the order specified for the assembly in 
    `the negspy repository <https://github.com/pkerpedjiev/negspy/tree/master/negspy/data>`_.

Ordering the chromosomes in the input file
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

    input_file=~/Downloads/phastCons100way.txt.gz;
    output_file=~/Downloads/phastConst100way_ordered.txt;
    chromnames=$(awk '{print $1}' ~/projects/negspy/negspy/data/hg19/chromInfo.txt);
    for chr in $chromnames; 
        do echo ${chr}; 
        zcat $input_file | grep "\t${chr}\t" >> $output_file;
    done;


Aggregation by addition
^^^^^^^^^^^^^^^^^^^^^^^

Assume we have an input file that has ``id chr start end value1 value2`` pairs::

    location        chrom   start   end     copynumber      segmented
    1:2900001-3000000       1       2900001 3000000 -0.614  -0.495
    1:3000001-3100000       1       3000001 3100000 -0.407  -0.495
    1:3100001-3200000       1       3100001 3200000 -0.428  -0.495
    1:3200001-3300000       1       3200001 3300000 -0.437  -0.495


We can aggregate this file by recursively summing adjacent values. We have to
indicate which column corresponds to the chromosome (``--chromosome-col 2``),
the start position (``--from-pos-col 3``), the end position (``--to-pos-col 4``) 
and the value column (``--value-col 5``). We specify that the first line
of the data file contains a header using the (``--has-header``) option.

.. code-block:: bash

    clodius aggregate bedgraph          \
        test/sample_data/cnvs_hw.tsv    \
        --output-file ~/tmp/cnvs_hw.hitile \
        --chromosome-col 2              \
        --from-pos-col 3                \
        --to-pos-col 4                  \
        --value-col 5                   \
        --assembly grch37               \
        --nan-value NA                  \
        --transform exp2                \
        --has-header                    

Data Transform
""""""""""""""

The dataset used in this example contains copy number data that has been log2
transformed. That is, the copy number given for each bin is the log2 of the
computed value. This is a problem for HiGlass's default aggregation method of
summing adjacent values since :math:`\log_2 a + \log_2 b \neq \log_2 ab`.

Using the ``--transform exp2`` option tells clodius to raise two to the
power of the provided value before doing the transformation and storing. As
an added benefit, NaN values become apparent in the resulting because they
have values of 0.

NaN Value Identification
""""""""""""""""""""""""

NaN (not a number) values in the input file can be specified using the
``--nan-value`` option.  For example, ``--nan-value NA`` indicates that
whenever *NA* is encountered as a value it should be treated as NaN. In the
current implementation, NaN values are simply treated as 0. In the future, they
should be assigned a special value so that they are ignored by `HiGlass`_.

.. _higlass: http://higlass.io

When NaN values are aggregated by summing, they are treated as 0 when added to
another number. When two NaN values are added to each other, however, the
result is Nan.

NaN Value Counting
""""""""""""""""""

Sometimes, we just want to count the number of NaN values in the file. The
``--count-nan`` option effectively treats NaN values as 1 and all other values
as 0. This makes it possible to display a track showing how many NaN values are
present in each interval. It also makes it possible to create compound tracks
which use that information to normalize track values.

bigWig files
------------

`bigWig files <https://genome.ucsc.edu/goldenpath/help/bigWig.html>`_ store
genomic data in a compressed, indexed form that allows rapid retrieval and
visualization. Unfortunately, for the time being, HiGlass does not support
displaying bigWig files and they must first be converted to hitile files.
This can be done using clodius's aggregate bigwig function:

.. code-block:: bash

    clodius aggregate bigwig input.bigWig \
        --output-file output.hitile \
        --assembly hg19

The resulting file can be loaded into HiGlass as described in the
:ref:`loading-into-higlass` section below.

Loading into HiGlass
^^^^^^^^^^^^^^^^^^^^

Too see .hitile-typed datasets in higlass, use the docker container to load them:

.. code-block:: bash

    docker exec higlass-container python \
            higlass-server/manage.py ingest_tileset \
            --filename /tmp/cnvs_hw.hitile \
            --filetype hitile \
            --datatype vector

It can also be loaded using a curl commands:

.. code-block:: bash
    
    curl -u `cat ~/.higlass-server-login`  \
        -F "datafile=@cnvs_hw.hitile" \
        -F "filetype=hitile" \
        -F "datatype=vector" \
        -F "coordSystem=hg19" \
        http://higlass.io:80/api/v1/tilesets/

.. todo:: And navigate to 127.0.0.1:8989, click on the '+' symbol, select a track
          position, find the dataset in the list of the datasets and click OK to
          view it. And stuff.

Cooler files
------------
`Cooler files <https://github.com/mirnylab/cooler>`_ (extension .cool) store 
arbitrarily large 2D genomic matrices, such as those produced via Hi-C and other high 
throughput proximity ligation experiments. HiGlass can render cooler files containing
matrices of the same dataset at a range of bin resolutions or *zoom levels*, so called multiresolution 
cool files (typically denoted .mcool).

From pairs
^^^^^^^^^^

Often you will start with a list of pairs (e.g. contacts, interactions) that need to be aggregated.
For example, the 4DN-DCIC developed a `standard pairs format <https://github.com/4dn-dcic/pairix/blob/master/pairs_format_specification.md>`_ for HiC-like data. However, you 
need only a tab-delimited file with columns representing ``chrom1``, ``pos1``, ``chrom2``, ``pos2``, optionally gzipped.

Currently, these need to be sorted and indexed with either pairix or tabix to be ingested into a cooler. You also need to
provide a list of chromosomes in semantic order (chr1, chr2, ..., chrX, chrY, ...) in a
two-column `chromsizes <https://github.com/pkerpedjiev/negspy/blob/master/negspy/data/hg19/chromSizes.tsv>`_ file.
For example, if ``chrom1`` and ``pos1`` are the first two columns, and ``chrom2`` and ``pos2`` are in columns 4 and 5:

.. code-block:: bash

    cooler csort -c1 1 -p1 2 -c2 4 -p2 5 mypairs.txt hg19.chrom.sizes

This will generate a sorted and compressed pairs file ``mypairs.blksrt.txt.gz`` along with a companion pairix ``.px2`` index file. To aggregate and ingest at a fixed resolution (e.g. 1kb) use the ``cload pairix`` command. 

.. code-block:: bash
    
    cooler cload pairix hg19.chrom.sizes:1000 mypairs.blksrt.txt.gz mycooler.1000.cool
    
This will be the *base resolution* for the multires cooler you will generate.

From a matrix
^^^^^^^^^^^^^
If your base resolution data is already aggregated, you can ingest data in one of two formats. Use ``cooler load`` to ingest.

1. COO: Sparse matrix upper triangle `coordinate list <https://en.wikipedia.org/wiki/Sparse_matrix#Coordinate_list_(COO)>`_ , i.e. tab-delimited sparse matrix triples (row_id, col_id, count). This is an output of pipelines like HiCPro.

.. code-block:: bash
    
    cooler load -f coo hg19.chrom.sizes:1000 mymatrix.coo.txt mycooler.1000.cool

2. BG2: A 2D "extension" of the `bedGraph <https://genome.ucsc.edu/goldenpath/help/bedgraph.html>`_ format. Tab delimited with columns representing ``chrom1``, ``start1``, ``end1``, ``chrom2``, ``start2``, ``end2``, ``count``. Currently, these require sorting and indexing just like pairs files (using start instead of pos columns). 

.. code-block:: bash
    
    cooler csort -c1 1 -p1 2 -c2 4 -p2 5 mymatrix.bg2 hg19.chrom.sizes
    cooler load -f bg2 hg19.chrom.sizes:1000 mymatrix.blksrt.bg2.gz mycooler.1000.cool
    
Zoomify
^^^^^^^
To recursively aggregate your matrix into a multires file, use the `zoomify` command.

.. code-block:: bash
    
    cooler zoomify mycooler.1000.cool

The output will be a file called `mycooler.1000.mcool` with zoom levels increasing by factors of 2. You can also 
request an explicit list of resolutions, as long as they can be obtained via integer multiples starting from the base resolution. HiGlass performs well as long as zoom levels don't differ in resolution by greater than a factor of ~5.

.. code-block:: bash

    cooler zoomify -r 5000,10000,25000,50000,100000,500000,1000000 mycooler.1000.cool
   
If this is Hi-C data or similar, you probably want to apply iterative correction (i.e. matrix balancing normalization) by using the ``--balance`` option.

Loading pre-zoomed data
^^^^^^^^^^^^^^^^^^^^^^^
If the matrices for the resolutions you wish to visualize are already available, you can ingest each one independently into the right location inside the file using the `Cooler URI <http://cooler.readthedocs.io/en/latest/api.html#uri-string>`_ ``::`` syntax.

HiGlass expects each zoom level to be stored at a location named ``resolutions/{binsize}``.

.. code-block:: bash

    cooler load -f bg2 hg19.chrom.sizes:1000 mymatrix.blksrt.bg2.gz mycooler.1000.cool::resolutions/1000
    cooler load -f bg2 hg19.chrom.sizes:5000 mymatrix.blksrt.bg2.gz mycooler.5000.cool::resolutions/5000
    cooler load -f bg2 hg19.chrom.sizes:10000 mymatrix.blksrt.bg2.gz mycooler.10000.cool::resolutions/10000
    ...

See the *cooler* `docs <http://cooler.readthedocs.io/>`_ for more information.

.. _loading-into-higlass:


Multivec Files
--------------

Multivec files store arrays of arrays organized by chromosome. To aggregate this
data, we need an input file where chromsome is a separate dataset. Example:

.. code-block:: python

    f = h5py.File('/tmp/blah.h5', 'w')

    d = f.create_dataset('chr1', (10000,5), compression='gzip')
    d[:] = np.random.random((10000,5))
    f.close()

This can be aggregated to multiple resolutions using `clodius aggregate multivec`:

.. code-block:: bash

    clodius aggregate multivec \
        --chromsizes-filename ~/projects/negspy/negspy/data/hg38/chromInfo.txt \
        --starting-resolution 1000 \
        --row-infos-filename ~/Downloads/sampled_info.txt \
        my_file_genome_wide_hg38_v2.multivec

The `--chromsizes-filename` option lists the chromosomes that are in the input
file and their sizes.  The `--starting-resolution` option indicates that the
base resolution for the input data is 1000 base pairs.

Development
-----------

In order ot display large datasets at multiple resolutions, HiGlass requires
the input data to be aggregated at multiple resolutions. For Hi-C data, this is
done using the `cooler <https://github.com/mirnylab/cooler/>`_ package. For
other types of data it is done using the `clodius
<https://github.com/hms-dbmi/clodius>`_ package.

Example
^^^^^^^

Let's say that we have a new datatype that we want to use in higlass. In this
example, we'll use HDF5 to store an array of length 100000 at multiple
resolutions. Each resolution will be half of the previous one and will be stored
as an array in the HDF5 file.

To begin, we'll create an HDF5 and add a group that will store the 
different resolutions:

.. code-block:: python

    import h5py
    import numpy as np

    f = h5py.File('/tmp/my_file.multires', 'w')

    f.create_group('resolutions')

We'll add our highest resolution data directly:

.. code-block:: python

    array_length = 100000 
    f['resolutions'].create_dataset('1', (array_length,))
    f['resolutions']['1'][:] = np.array(range(array_length))

    print(f['resolutions']['1'][:5])

This should output ``[ 0.  1.  2.  3.  4.]``. Next we want to recursively
aggregate this data so that adjacent entries are summed. This can be
accomplished in a concise manner using numpy's ``.resize(-1,2).sum(axis=1)``
function. 

How many recursions do we need? In the end, we want to be able to fit all
the data into one 1024 element tile. This means that after n aggregations,
we need to have less than 1024 elements. 

.. code-block:: python

    tile_size = 1024
    max_zoom = math.ceil(math.log(array_length / tile_size) / math.log(2))




Building for development
^^^^^^^^^^^^^^^^^^^^^^^^

The recommended way to develop ``clodius`` is to use a `conda`_ environment and 
install ``clodius`` with develop mode:

.. _conda: https://conda.io/docs/intro.html

.. code-block:: bash

    python setup.py develop

Note that making changes to the ``clodius/fast.pyx`` `cython`_ module requires an
explicit recompile step:

.. _cython: http://docs.cython.org/en/latest/src/quickstart/cythonize.html

.. code-block:: bash
   
    python setup.py build_ext --inplace

Testing
^^^^^^^

The unit tests for clodius can be run using `nosetests`_::

    nosetests tests

Individual unit tests can be specified by indicating the file and function
they are defined in::

    nosetests test/cli_test.py:test_clodius_aggregate_bedgraph


.. _nosetests: http://nose.readthedocs.io/en/latest/



Building the documentation
^^^^^^^^^^^^^^^^^^^^^^^^^^

Building the documentation from the root directory is a matter of running
``sphinx-build``::

    sphinx-build docs -b html docs/_build/html

To view the documentation, go to the build directory and start an http server::

    cd docs/_build/html
    python -m http 8081
