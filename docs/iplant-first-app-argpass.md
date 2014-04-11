Crafting applications using Agave argument passing
==================================================
Agave now supports limited forms of automatic command line generation. We will demonstrate how to take advantage of it.

Background
----------
One great challenge in wrapping command line programs in abstraction layers is the incredible diversity in the way they are parameterized. Some use --long flags, some use Java-style KEY=VALUE pairs, some use config.files, and so on. iPlant's Discovery Environment provides a sophisticated, fully-automated command line builder, while previous releases of Agave required that you painstakingly handle every command flag in your template script. The former approach is great when it works, and leads to rapid application development, but is a challenge for a solid 40% of applications. The latter approach provides perfect flexibility at the price of requiring the app developer to write a lot of (often repetitive) conditional shell script code. Agave V2 includes new features to help streamline this process. To use them, you need to make subtle changes to your template script and modify your input, parameter, and output definitions a bit. 

Technical Details
-----------------
When the Agave service creates an execution script on the executionSystem, it takes the app's template file and performs find-and-replace substitutions, replacing variables matching input.id, parameter.id, and output.id in the template with values passed as part of the job invocation process. The original implementation of Agave dropped in just the value, and it was up to developers to specify the command flag. Now, you may specify, in your app description, a value for "input.details.argument" indicating (if needed) a leading command flag. If you then specify *"input.details.showArgument": true*, when the execution script is created it will replace ${input.id} in the template with "argument value" instead of just "value". The same applies to parameters. Here's a simple example to illustrate how to craft a specific command line:

```sh
# Here's a silly command that we want to run as part of our job
a.out --input foobaz.txt --beastmode --over 9000 --powerlevel --dragonball=Z > stdout.txt
```

### Template file
```sh
ARGS="--beastmode ${keyvalParam} ${flagParam}"
if [ -n "${manualKeyvalParam}" ]; then ARGS="$ARGS --dragonball=${manualKeyvalParam}"
a.out ${inputFile} $ARGS > stdout.txt
```

### Input specification
```json
{"inputs":[
    {"id":"inputFile",
     "value":
        {"default":"shared/foobaz.txt",
         "order":0,
         "required":true,
         "validator":".txt$",
         "visible":true},
     "semantics":
        {"ontology":["http://sswapmeet.sswap.info/mime/text/Plain"],
         "minCardinality":1,
         "fileTypes":["raw-0"]},
     "details":
        {"description":"",
         "label":"The Text File",
         "argument":"--input ",
         "showArgument":true}}]}
```

### Parameters
```json
{"parameters":[
    {"id":"keyvalParam",
     "value":
        {"default":"9000",
         "order":1,
         "required":true,
         "type":"number",
         "validator":"",
         "visible":true},
     "semantics":
        {"ontology":["xs:integer"]},
     "details":
        {"description":null,
         "label":"An innumerable quantifier",
         "argument":"--over ",
         "showArgument":true}},
	{"id":"manualKeyvalParam",
     "value":
        {"default":"Z",
         "order":1,
         "required":false,
         "type":"string",
         "validator":"",
         "visible":true},
     "semantics":
        {"ontology":["xs:string"]},
     "details":
        {"description":"",
         "label":"Which Dragonball are you?",
         "argument":"",
         "showArgument":false}},
    {"id":"flagParam",
     "value":
        {"default":false,
         "order":1,
         "required":false,
         "type":"bool",
         "validator":"",
         "visible":true},
     "semantics":
        {"ontology":["xs:boolean"]},
     "details":
        {"description":null,
         "label":"Refers to power level",
         "argument":"--powerlevel",
         "showArgument":true}}]}
```

### Resulting execution script
```sh
ARGS="--beastmode --over 9000 --powerlevel  --dragonball=Z"
a.out --input foobaz.txt ${ARGS} > stdout.txt
```

### Explanatory notes
1. Use of the "flag" type for flagParam: If the value at run time is true, the argument value is passed in and used for authoring the execution script, otherwise it is not. 
2. Static ARGS: We always want to run a.out in Beast Mode. This could be modeled as an immutable, invisible parameter, but can also just be hard-coded into the ARGS string.
3. Mixing passed-arguments with manually-handled parameters: This use case is illustrated by the handling of our "dragonball" parameter.
4. Note the trailing spaces on arguments for inputFile and keyvalParam. Agave concatenates argument and value without automatically adding any whitespace. This allows you to construct arguments that look like --ion_cannon=low-orbit

Implementation: "samtools sort" using argument passing
-------------------------------------------------------



*This completes the section on using Agave argument passing in your apps.*

[Back to READ ME](../README.md)
