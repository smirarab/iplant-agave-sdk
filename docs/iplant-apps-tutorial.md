Deploying new iPlant applications via Agave API
===============================================

Overview
--------
In January 2014, version 2 of the iPlant Agave API was released, bringing with it some powerful new features. This tutorial provides a walk-through for deploying your own applications. It's targeted to iPlant developers, though others can easily use it with some adaptation. Instructions to _type_ can alternatively be executed as a _copy and paste._

The tutorial assumes that you have already registered an iPlant account and have access to a computer system, e.g. Stampede. You will need your account usernames and passwords for iPlant and Stampede. You will also need the path to your $WORK directory on TACC systems.

## Installing the iPlant API software development kit

The API comes bundled with a set of command line scripts and example files to help make you more productive. Using the command line scripts is generally easier than hand-crafting cURL commands, but if you prefer that route, consult [http://agaveapi.co/getting-started-with-the-agave-api/].

Using a terminal interface, _ssh_ into the system you will be working with (e.g. Stampede, Lonestar, etc)

```sh
ssh stampede.tacc.utexas.edu
# Check out the SDK from the iPlant's GitHub
# Load an updated git module by typing:
module load git
## Clone the SDK repository:
git clone https://github.com/iPlantCollaborativeOpenSource/iplant-agave-sdk.git
## Add SDK scripts to your PATH:
echo "IPLANT_SDK_HOME=$PWD/iplant-agave-sdk" >> ~/.bashrc
echo "PATH=\$PATH:\$IPLANT_SDK_HOME/scripts:" >> ~/.bashrc
echo "PATH=\$PATH:\$IPLANT_SDK_HOME/foundation-cli/bin" >> ~/.bashrc
## To re-init bash _type_:
source ~/.bashrc
```
*This completes the section on installing the iPlant Agave SDK.*

Creating a client and getting a set of API Keys
-----------------------------------------------

The Agave API uses OAuth 2 for managing authentication and authorization. Before you start working with the API, you will need to create a OAuth client application which is associated with a set of API keys. This is a one-time action, so if you already have your API keys, skip to the next section. If not, you can create your keys using the Clients service as follows:

```sh
clients-create -S -v -N my_client -D "Client app used for scripting up cool stuff"
```

### Notes
The \-N flag allows you to specify a machine-readable name for your application; \-D provides the description, and \-S option stores your API keys for future use, so you will not need to manually enter them when you authenticate later.

You should get a response from clients-create that looks similar to this:
```json
{
    "_links": {
        "self": {
            "href": "https://agave.iplantc.org/clients/v2/my_client"
        },
        "subscriber": {
            "href": "https://agave.iplantc.org/profiles/v2/vaughn"
        },
        "subscriptions": {
            "href": "https://agave.iplantc.org/clients/v2/my_client/subscriptions/"
        }
    },
    "callbackUrl": "",
    "consumerKey": "fMYfhijQE8ajqcKaswlGH4D5sngh",
    "consumerSecret": "8FZ6K9QwN1PY6bA9SOcyi4oaEkMa",
    "description": "Client used for app development",
    "name": "my_client",
    "tier": "Unlimited"
}
```
Although much of the process of interacting with the Agave API is automated, you may need access to the consumerKey and consumerSecret for other types of OAuth2-based interaction. If you ever need to retrieve the API keys for a particular client application, you can always do so at https://agave.iplantc.org/store/site/pages/subscriptions.jag

*This completes the section on obtaining a set of API keys.*

Obtaining an OAuth 2 authentication token
--------------------------------------------

In order to interact with iPlant API services, you will need to acquire a authentication token, which is tied to the client application you have created. The command for accomplishing this is:
```sh
# From your terminal interface, _type_:
auth-tokens-create -S -v
```
* You will first be prompted to enter your *Consumer Secret*. Copy your consumerSecret from above, where you created a client application, and _paste_ it in your terminal interface where prompted.
* You will next be prompted to enter your *Consumer Key*. _Copy_ your *consumerKey *and _paste_ this in your terminal interface where prompted.
* You will then be prompted to enter your *Agave tenant username*. _Type_ your iPlant username.
* You will then be prompted to enter your *Agave tenant password*. _Type_ your iPlant password.
* At this point, you should receive an affirmation of success in your terminal that resembles this one:
```json
Token successfully refreshed and cached for 14400 seconds
{
    "access_token": "e431846eb2c917a4c5796eb1cc2c6f",
    "expires_in": 14400,
    "refresh_token": "8212a515b26ebc1aec5d6e232bb455b",
    "token_type": "bearer"
}
```

If your token at some point in time expires, simply _type_ the same auth-tokens-create command. You'll only need to enter your iPlant password, as the other values will be automatically remembered. You will need to configure the iPlant API SDK on each system you plan to develop on. To do so, clone the repo from GitHub, update your environment variables as above, then run _auth-tokens-create -S_ using the consumerSecret and consumerKey for your Oauth2 client.

More information on this step is available at [http://agaveapi.co/authentication-token-management/]

*This completes the section on obtaining an OAuth2 authentication token.*

Setting up development systems
------------------------------

One of the most interesting features of the new Agave release is its ability to make use of diverse systems for storing data and executing applications. This offers both an opportunity and a challenge if you come to us from Agave V1 application development. Under that model, there were a limted number of monolithic, shared public systems; you had to get special permission to do app development; and debugging was painful because your jobs didn't run under your username on the host systems. We will now set up private versions of the public systems that iPlant uses to power Discovery Environment applications. You can do this manually via the systems-clone and systems-list commands, but we have bundled an automated script with this SDK to make it even more straightforward. You only need to do this once, as the systems created here will be associated with your iPlant credentials no matter where you are developing in the future. 

First, gather up your iPlant credentials (specifically your password), your TACC username and password, and the full path to your WORK directory on TACC systems. To find out this last bit of information:
```sh
ssh stampede.tacc.utexas.edu 'echo $WORK'
```

Now, _type_ _iplant-systems-create_ which will walk through a set of templates stored in $IPLANT_SDK_HOME and create a new, private version of each TACC system. Here's an example log from running this for myself:

```sh

We are going to create and enroll personal copies of various HPC systems
used by iPlant staff developers to build apps.
The following assumes you have created an Agave client via client-create.
We will now ask you to refresh your access token...

API secret [6bA9SOcyi4o8FZ6K9QwN1PYaEkMa]: 
API key [jqcKaswlGH4D5snghfMYfhijQE8a]: 
API username [vaughn]: 
API password: 
Token successfully refreshed and cached for 13150 seconds
ef5f619c98ca45c56d4bc2a29dbc723b

We will now configuring TACC-managed systems and need some info from you
    TACC username
	TACC password
	TACC Project name
	The path to your $WORK directory on TACC systems

Do you have all the information required? [Yes]: 
OK. Let's begin.
Enter your TACC user account []: vaughn
Confirmed: TACC user account is vaughn
Enter your TACC account password []: \n
Enter a TACC project associated with this system [iPlant-Collabs]: 
Confirmed: TACC project is iPlant-Collabs
Enter your TACC work directory []: /work/01374/vaughn
Confirmed: TACC work directory is 
tacc-lonestar4-template
Enrolling private system tacc-lonestar4-template
Successfully added system lonestar4-04012014-1718-vaughn
tacc-maverick-template
Enrolling private system tacc-maverick-template
Successfully added system maverick-04012014-1718-vaughn
tacc-stampede-template
Enrolling private system tacc-stampede-template
Successfully added system stampede-04012014-1718-vaughn

Here is a listing of your systems:
lonestar4.tacc.teragrid.org
lonestar4-04012014-1718-vaughn
stampede.tacc.utexas.edu
stampede-04012014-1718-vaughn
maverick-04012014-1718-vaughn
data.iplantcollaborative.org
```

In the current working directory, you will find several JSON files. These contain the descriptions of the private systems created, including your password for each system. Either delete them or set their permissions so that only you can read them:
```sh
chmod 600 *.json
```

Verify that you can access the private systems (the ones containing your username) as follows:
_files-list -S maverick-04012014-1718-vaughn_ where -S specifies the system. This should display a listing of your $WORK directory on the specified TACC system. 

*This completes the section on setting up private iPlant execution systems.*

Registering your first application
----------------------------------

The previous steps in this tutorial were one-time only affairs. You can start *here* if you have set everything up and are just interested in app development. 

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
cd iPlant/src
wget "http://downloads.sourceforge.net/project/samtools/samtools/0.1.19/samtools-0.1.19.tar.bz2"
tar -jxvf samtools-0.1.19.tar.bz2
cd samtools-0.1.19
make CC=icc
# Copy the samtools binary and support scripts to the project directory
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
# Package up the bin directory as an archive and remove the original         
tar -czf bin.tgz bin && rm -rf bin
```
