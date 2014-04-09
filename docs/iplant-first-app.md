Creating an iPlant application for TACC Stampede
================================================
Agave API apps have a generalized structure that allows them to carry dependencies around with them. In the case below, _package-name-version.dot.dot_ is a folder that you build on your local system, then store in the iPlant Data Store in a designated location (we recommend /iplant/home/USERNAME/applications). It contains binaries, support scripts, test data, etc. all in one package. Agave basically uses a very rough form of containerized applications (more on this later). Thus, all apps should look something like the following, no matter what code you are working to deploy:

```sh
package-name-version.dot.dot
|--system_name
|----bin.tgz (optional)
|----test.sh
|----template.(sge|slurm|pbs)
|----test_data (optional)
|----app.json
```

When a job is executed, this bundle of files is staged into place in a temporary directory on the target system. Then, the input data files (we'll show you how to specify those are later) are staged into place. Next, Agave writes a scheduler submit script (using a template you provide i.e. template.xxx) and puts it in the queue on the target system. It monitors progress of the job and assuming it completes, Agave copies all new files to the location specified when the job was submitted. Along the way, critical milestones are recorded in the job's history. 

Create an application bundle
----------------------------
We will now go through the process of building and deploying an Agave application to provide 'samtools sort' functionality on the Stampede system. The following steps assume you have properly installed and configured the iPlant SDK on Stampede.

```sh
# Log into Stampede and set up a project directory
ssh stampede.tacc.utexas.edu
cd $WORK
mkdir iPlant
mkdir iPlant/src
mkdir -p iPlant/samtools-0.1.19/stampede/bin
mkdir -p iPlant/samtools-0.1.19/stampede/test
# Build samtools using the Intel C Compiler
# If you don't have icc, gcc will work but icc gives faster code
cd iPlant/src
wget "http://downloads.sourceforge.net/project/samtools/samtools/0.1.19/samtools-0.1.19.tar.bz2"
tar -jxvf samtools-0.1.19.tar.bz2
cd samtools-0.1.19
make CC=icc
# Copy the samtools binary and support scripts to the project bin directory
cp -R samtools bcftools misc ../../samtools-0.1.19/stampede/bin/
cd ../../samtools-0.1.19/stampede
# Test that samtools will launch
bin/samtools

Program: samtools (Tools for alignments in the SAM format)
Version: 0.1.19-44428cd

Usage:   samtools <command> [options]

Command: view        SAM<->BAM conversion
         sort        sort alignment file
         mpileup     multi-way pileup...
# Package up the bin directory as an compressed archive 
# and remove the original. This preserves the execute bit
# over file movements and consolidates movement of all
# bundled dependencies in bin to a single operation
tar -czf bin.tgz bin && rm -rf bin
```
Create a test script
--------------------
Your objective here is to create a script that you know will run to completion under the Stampede scheduler and environment. The standard is to use Bash for these, so that's what we'll do. You need to accomplish a few things in your script:
* Unpack binaries from bin.tgz
* Extend your PATH to contain bin
* Craft some option-handling logic to accept parameters from Agave
* Clean up when you're done

First, grab some test data into your working directory. You can use a test file
```sh
files-get -S data.iplantcollaborative.org shared/iplantcollaborative/example_data/Samtools_mpileup/ex1.bam
```
or you can use any other BAM file for your testing purposes. Just change the filename in your script accordingly.

Now, author your script. You can paste this script into a file called *test-sort.sh* or you can copy it from $IPLANT_SDK_HOME/examples/samtools-0.1.19/stampede/test-sort.sh

```sh
#!/bin/bash
 
# Agave automatically writes these scheduler
# directives when you submit a job but we have to
# do it by hand when writing our test

#SBATCH -p development
#SBATCH -t 00:30:00
#SBATCH -n 15
#SBATCH -A iPlant-Collabs 
#SBATCH -J test-samtools
#SBATCH -o test-samtools.o%j

# Set up inputs and parameters
# We're emulating passing these in from Agave
#
#inputBam is the name of one of the inputs we
#will specify later
inputBam="ex1.bam"
# outputPrefix is a parameter we specify later
# For now, we will set it statically
outputPrefix=sorted
# maximum memory for sort, in bytes
maxMemSort=500000000
# Boolean: Sort by name instead of coordinate
nameSort=0

# Unpack the bin.tgz file containing samtools binaries
# If you are relying entirely on system-supplied binaries you don't
# need this bit
tar -xvf bin.tgz
# Set the PATH to include binaries in bin
export PATH=$PATH:"$PWD/bin"
 
# Build up an ARGS string for the program
# Start with empty ARGS...
ARGS=""
# Add -m flag if maxMemSort was specified
if [ ${maxMemSort} > 0 ]; then ARGS="${ARGS} -m $maxMemSort"; fi
ARGS="-m $maxMemSort"
# Boolean handler for -named sort
if [ ${nameSort} == 1 ]; then ARGS="${ARGS} -n "; fi
 
# Run the actual program
samtools sort ${ARGS} $inputBam ${outputPrefix}

# Now, delete the bin/ directory
rm -rf bin
```
Submit the job to the queue on Stampede...
```sh
sbatch test-sort.sh 
```
Assuming you didn't make any typos, you'll end up with a sorted BAM called *sorted.bam*, and your bin directory (but not the bin.tgz file) should be fone. Congratulations, you're in the home stretch: it's time to turn the test script into an Agave script template.

Creating an application description
-----------------------------------
In order for Agave to know how to run an instance of the application, we need to provide quite a bit of metadata about the application. This includes a unique name and version, the location of the application bundle, the identities of the execution system and destination system for results, whether its an HPC or other kind of job, the default number of processors and memory it needs to run, and of course, all the inputs and parameters for the actual program. It seems complicated, but only because you're comfortable with the command line already. Agave's model lets applications be portable across systems and present a rationalized interface to consumers so that they can focus on science rather than fighting with compiler flags and the vagaries of learning every wierd command flag across all the applications they want to use in a workflow.

Rather than have you write a description for "samtools sort" yourself, we're going to systematically dissect an existing file provided with the SDK. Go ahead and copy the file into place and open it in your text editor of choice. If you don't have the SDK installed, you can [grab it here](../examples/samtools-0.1.19/stampede/samtools-sort.json).
```sh
cd $WORK/iPlant/samtools-0.1.19/stampede/
cp $IPLANT_SDK_HOME/examples/samtools-0.1.19/stampede/samtools-sort.json .
```

### Overview

The file *samtools-sort.json* is a [JSON](http://www.json.org/) file written to conform to a specific data model. You can find fully fleshed out details about all fields in this document in the Model tab of the [Agave API live docs on the /apps service](http://agaveapi.co/live-docs/#!/apps/add_post_1), but we will dive into a few key details here. *To make this file work for you, you will at minimum need to edit its executionSystem to match your private instance of Stampede*.

All Agave application descriptions have the following structure:

```json
{	"application_metadata":"value",
	"inputs":[],
	"parameters":[],
	"outputs":[]
}
```

There is a defined list of application metadata fields, some of which are mandatory. Inputs, parameters, and outputs are specified as an array of specific data structures, which will be described below.

### Application metadata

| Field | Mandatory | Type | Description |
| ----- | --------- | ---- | ----------- |
| checkpointable | X | boolean | Application supports checkpointing |
| defaultMemoryPerNode | | integer | Default RAM (GB) to request per compute node |
| defaultProcessorsPerNode | | integer | Default RAM (GB) to request per compute node |
| defaultMaxRunTime | | integer | Default RAM (GB) to request per compute node |
| defaultNodeCount | | integer | Default number of compute nodes per job |
| defaultQueue | | string | On HPC systems, default batch queue for jobs |
| deploymentPath | X | string | Path relative to homeDir on deploymentSystem where application bundle will reside |
| deployementSystem | X | string | The Agave-registered STORAGE system upon which you have write permissions where the app bundle resides |
| executionSystem | X | string | An Agave-registered EXECUTION system upon which you have execute and app registration permissions where jobs will run |
| helpURI | X | string | A URL pointing to help or description for the app you are deploying |
| label | X | string | Human-readable title for the app |
| longDescription | | string | A short paragraph describing the functionality of the app |
| modules | | array[string] | Ordered list of modules on systems that use lmod or modules |
| name | X | string | unique URL-compatible (no special chars or spaces) name for the app |
| ontology | X | array[string] | List of ontology terms (or URIs pointing to ontology terms) associated with the app |
| parallelism | X | serial|parallel | Is your application capable of using more than a single compute node? |
| shortDescription | X | string | Brief description of the app |
| storageSystem | X | string | The Agave-registered STORAGE system upon which you have write permissions. Default source of and destination for data consumed and emitted by the app |
| tags | | array[string] | List of human-readable tags for the app |
| templatePath | X | string | Path to the shell template file, relative to deploymentPath |
| testPath | X | string | Path to the shell test file, relative to deploymentPath |
| version | X | string | Preferred format: Major.minor.point integer values for app |

### Inputs

To tell Agave what files to stage into place before job execution, you need to define the app's inputs. In our example, we need to tell Agave that a BAM file is needed to act as the subject of our sort. Here's one way of making such a specification:

```json
   {"id":"inputBam",
     "value":
        {"default":"",
         "order":0,
         "required":true,
         "validator":"",
         "visible":true},
     "semantics":
        {"ontology":["http://sswapmeet.sswap.info/mime/application/X-bam"],
         "minCardinality":1,
         "fileTypes":["raw-0"]},
     "details":
        {"description":"",
         "label":"The BAM file to sort",
         "argument":null,
         "showArgument":false}}]
```

Here's a walkthrough of what these fields mean:

| Field | Mandatory | Type | Description |
| ----- | --------- | ---- | ----------- |
| id | X | string | This is the "name" of the file. You will use this in your wrapper script later whenever you need to refer to the BAM file being sorted |
| id.value.default | | string | The path, relative to X, of the default value for the input |
| id.value.order | | integer | Ignore for now |
| id.value.required | X | boolean | Is specification of this input mandatory to run a job? |
| id.value.validator | | string | Java-format regular expression to restrict valid values |
| id.value.visible | | boolean | When automatically generated a UI, should this field be visible to end users? |
| semantics.ontology | | array[string] | List of ontology terms (or URIs pointing to ontology terms) applicable to the input format |
| semantics.minCardinality | | integer | Minimum number of values accepted for this input |
| semantics.maxCardinality | | integer | Maximum number of values accepted for this input |
| semantics.fileTypes | X | array[string] | List of Agave file types accepted. Always use "raw-0" for the time being |
| details.description | | string | Human-readable description of the input. Often implemented as contextual help in automatically generated UI |
| details.label | | string | Human-readable label for the input. Often implemented as text label next to the field in automatically generated UI |
| details.argument | | string | The command-line argument associated with specifying this input at run time |
| details.showArgument | | boolean | Include the argument in the substitution done by Agave when a run script is generated |

*A note on paths*: In this iPlant tutorial, we assume for the most part that you will stage data to and from the default storage system for iPlant users "data.iplantcollaborative.org". In this case, you can use relative paths relative to homeDir on that system (i.e. vaughn/analyses/foobar). For added portability, or to marshal data from other storageSystems or public URIs, you can always specify fully qualified URIs in one of two forms:
* storageSystem namespace: agave://
* public URI namespace: https://www.cnn.com/index.html

### Parameters
### Outputs

Create a template script
------------------------
Make a copy of your test-sort.sh script, so you can refer back to it later.

```sh
cp test-sort.sh sort.template
```

Now, open template.slurm in the text editor of your choice. Delete the bash shebang line and the SLURM pragmas. Replace the hard-coded values for inputs and parameters with variables defined by your app description.

_NOTE TO SELF: Add comments and update script to illustrate use of Agave parameter passing function_

```sh
# Set up inputs...
# Since we don't check these when constructing the
# command line later, these will be marked as required
inputBam=${inputBam}
# and parameters
outputPrefix=${outputPrefix}
# Maximum memory for sort, in bytes
# Be careful, Neither Agave nor scheduler will
# check that this is a reasonable value. In production
# you might want to code min/max for this value
maxMemSort=${maxMemSort}
# Boolean: Sort by name instead of coordinate
nameSort=${nameSort}

# Unpack the bin.tgz file containing samtools binaries
tar -xvf bin.tgz
# Set the PATH to include binaries in bin
export PATH=$PATH:"$PWD/bin"
 
# Build up an ARGS string for the program
# Start with empty ARGS...
ARGS=""
# Add -m flag if maxMemSort was specified
if [ ${maxMemSort} > 0 ]; then ARGS="${ARGS} -m $maxMemSort"; fi
ARGS="-m $maxMemSort"
# Boolean handler for -named sort
if [ ${nameSort} == 1 ]; then ARGS="${ARGS} -n "; fi
 
# Run the actual program
samtools sort ${ARGS} $inputBam ${outputPrefix}

# Now, delete the bin/ directory
rm -rf bin
```
### Upload the application bundle

Each time you (or another user) requests an instance of samtools sort, Agave copies data from a "deploymentPath" on a "storageSystem" to create the temporary directory on an "executionSystem". Now that you've crafted the application bundle's dependencies and script template, it's time to store it somewhere accessible by Agave. 

*Note* If you've never deployed an Agave-based app, you may not have an applications directory in your home folder. Since this is where we recommend you store the apps, create one.
```sh
# Check to see if you have an applications directory
files-list -S data.iplantcollaborative.org IPLANTUSERNAME/applications
# If you see: File/folder does not exist
# then you need to create an applications directory
files-mkdir -S data.iplantcollaborative.org -N "applications" IPLANTUSERNAME/
```

Now, upload the application bundle:
```sh
# cd out of the bundle
cd $WORK/iPlant
# Upload using files-upload
files-upload -S data.iplantcollaborative.org -F samtools-0.1.19 IPLANTUSERNAME/applications
```
Any time you need to update the binaries, libraries, templates, etc. in your non-public application, you can just make the changes locally and re-upload the bundle. The next time Agave invokes a job using this application, it will stage out the updated version of the application bundle.

Post your app description
-------------------------

```sh
apps-addupdate -F samtools-0.1.19/stampede/samtools-sort.json
```

Any time you need to update the metadata description of your non-public application, you can just make the changes locally to the JSON file and and re-post it as such. The field *samtools-sort-0.1.19* at the end is the appid you're updating. Agave tries to guess from the JSON file but to remove uncertainty, we recommend always specifying it explicitly. 

```sh
apps-addupdate -F samtools-0.1.19/stampede/samtools-sort.json samtools-sort-0.1.19
```

The next time Agave invokes a job using this application, it will use the new description.
