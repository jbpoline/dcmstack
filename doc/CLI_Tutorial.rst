CLI Tutorial
============

DcmStack has two command line interfaces: *dcmstack* and *nitool*. The 
*dcmstack* command is used for converting DICOM data to Nifti files with 
the optional DcmMeta extension.  The *nitool* command is used to work 
with these exteneded Nifti files.

Advanced Conversion
-------------------

While the *dcmstack* command has many options, the defaults should do 
the right thing in most scenarios. To see a complete list of the command 
line options (with brief descriptions) use the *-h* option.

Embedding Meta Data
^^^^^^^^^^^^^^^^^^^

If the *--embed* option is used, all of the meta data in the source DICOM 
files will be extracted and summarized into a DcmMeta extension, which is 
then embedded into the Nifti header. The meta data keys are the keywords 
from the DICOM standard. For details on the DcmMeta extension see 
:doc:`DcmMeta_Extension`.

The meta data is filtered using regular expressions to minimize the chance 
of including any PHI (Private Health Information). There are two types of 
regular expressions used for filtering: 'exclude' and 'include' expressions. 
Any meta data where the key matches an exclude expression will be excluded,
**unless** it also matches an include expression. That is to say that the 
include expressions override the exclude expressions.

To see the list of the default regular expressions use the *--default-regexes*
option. To add an additional exclude expression use *--exclude-regex* (*-e*) 
option and to add an additional include expression use the *--include-regex* 
(*-i*) option.

By default, any private DICOM elements are ignored unless there is a 
"translator" for that element. To see a list of available translators use the 
*--list-translators* (*-l*) option. To disable a specific translator use the
*--disable-translator* option. To include private elements that don't have 
a translator use the *--extract-private* option.


Output Names and Grouping
^^^^^^^^^^^^^^^^^^^^^^^^^

The output name is determined by one or more Python format strings. The 
format strings are formatted with the meta data attributes for each input 
file, and every file with the same resulting output name is grouped 
together into a stack. 

The output name is primarily determined by the *--output-format* option. 
The default output format is '%(SeriesNumber)03d-%(ProtocolName)s'. 

There is also the *--opt-suffix* option. The result of this format string 
will be appended to the result of the *--output-format* option if and 
only if it differs between input DICOM files. The default optional suffix 
is 'TE_%(EchoTime).3f' which will produce multiple outputs for multi echo 
series.

Ordering Time and Vector Dimensions
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In addition to the three spatial dimensions, Nifti images can have time and 
(less commonly) vector dimensions. By default, if there is more than one 
volume they will be ordered along the time dimension using 'AcquisitionTime'. 
If this is not desired, a different attribute can be provided using the 
*--time-var* (*-t*) option. If you want to create a vector dimension use the 
*--vector-var* (*-v*) option followed by the attribute to use to order the 
volumes along that dimension.

If there isn't an attribute that can be used with a simple ascending order to 
sort along these dimensions, the *--time-order* or *--vector-order* options 
can be used. The argument to the option should be a text file with one value 
per line corresponding to the sorted order to use. 

Creating Uncompressed Niftis
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default the output Nifti files will be compressed, and thus have the 
extension '.nii.gz'. Almost every program that can read Nifti files will still 
read them if they are compressed. To override this behavior you can use the 
*--output-ext* option. 

Handling Bad Data
^^^^^^^^^^^^^^^^^

Valid DICOM files should have a specific preamble (an initial byte pattern) to 
identify them as a DICOM file. It is not uncommon to come across files that are 
missing this preamble but are otherwise valid (generally due to bad software). 
You can force dcmstack to try to read these files using the *--force-read* 
option.

With some data sets (generally EPI) slices can be missing their pixel data due 
to an error in the reconstruction. Using the *--allow-dummies* option will 
allow these files and fill the corresponding slice with the maximum possible 
value (i.e. 65535 for uint16).

Voxel Order
^^^^^^^^^^^

While the affine transform stored in the Nifti allows a mapping from voxel 
indices to patient space, some programs do not make use of the affine 
information. To provide a similar orientation in these programs we reorder 
voxels in the same manner as dcm2nii. This results in the row, column and 
slice indices starting from the most right, posterior, inferior corner 
relative to patient space. This can be overridden with the *--voxel-order* 
option. 

Working with Extended Nifti Files
---------------------------------

The *nitool* command can be used to perform various tasks with the extended 
Nifti files (that is files with the the DcmMeta extension embedded). The 
*nitool* command exposes functionality through a number of sub commands. 
To see a list of sub commands with brief explanations use the *-h* option.
To see detailed help for a specific subcommand use:

.. code-block:: console
    
    $ nitool <sub_command> -h

Looking Up Meta Data
^^^^^^^^^^^^^^^^^^^^

To lookup meta data in an extended Nifti, use the *lookup* sub command. If 
you don't specify a voxel index (using *--index*) then only constant meta 
data will be considered.

.. code-block:: console
    
    $ nitool lookup InversionTime 032-MPRAGEAXTI900Pre/032-MPRAGE_AX_TI900_Pre.nii.gz 
    900.0
    $ nitool lookup InstanceNumber 032-MPRAGEAXTI900Pre/032-MPRAGE_AX_TI900_Pre.nii.gz 
    $ nitool lookup InstanceNumber --index 0,0,0 032-MPRAGEAXTI900Pre/032-MPRAGE_AX_TI900_Pre.nii.gz 
    1
    $ nitool lookup InstanceNumber --index 0,0,1 032-MPRAGEAXTI900Pre/032-MPRAGE_AX_TI900_Pre.nii.gz 
    2

In the above example 'InversionTime' is contant across the Nifti and so an 
index is not required. The 'InstanceNumber' is not constant (it varies over 
slices) and thus only returns a result if an index is provided.

Merging and Splitting
^^^^^^^^^^^^^^^^^^^^^

To merge or split extended Nifti files use the *merge* and *split* sub 
commands. This will automatically create appropriate DcmMeta extensions for 
the output Nifti file(s). Both sub commands take a *--dimension* (*-d*) option 
to specify the index (zero based) of the dimension to split or merge along. 

If the dimension is not specified to the *split* command, it will use the last 
dimension (vector, time, or slice). By default each output will have the same 
name as the input only with the index prepended (zero padded to three spaces). 
A format string can be passed with the option *--output-format* (*-o*) to 
override this behavior.

If the dimension is not specified for the *merge* command, it will use the last 
singular or missing dimension (slice, time, or vector). By default the inputs 
will be merged in the order they are provided on the command line. To instead 
sort the inputs using some meta data key use the *--sort* (*-s*) option.

Dumping and Embedding
^^^^^^^^^^^^^^^^^^^^^

The DcmMeta extension can be dumped using the *dump* sub command. If no 
destination path is given the result will print to stdout. A DcmMeta extension 
can be embedded into a Nifti file using the *embed* sub command. If no input 
file is given it will be read from stdin. For details about the DcmMeta 
extension see :doc:`DcmMeta_Extension`.


