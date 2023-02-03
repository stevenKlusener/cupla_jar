# Cupla: An improved code duplication analysis 

## Introduction

Since several decades software quality has been an ongoing issue, it has been generally understood that *technical debt* (i.e., a low level of software quality) is a liability for an organisation leading to higher and less predictable maintenance costs. Nowadays it is quite standard to adopt tooling within the sofware delivery process (such as CI/CD) that applies a number of checks on the source code.

One of the most important checks is the detection of duplicated code, to avoid/reduce the following:
- Duplicated code also means duplicated maintenance;
- If duplication is not known (which is the case most of the time), and a bug is resolved in one place, the bug will still remain at the other place, leading to negative surprises over time.

Duplication can be avoided/resolved by applying software engineering techniques like object orientation to add proper abstractions (classes, methods/functions, etc) that can be reused. A small amount of duplication is acceptable, it is generally agreed that the duplication ratio should not exceed 5%.

Classically duplicated code fragments, called *clones*, are reported when they have a length of 100 tokens or more. A *token* is a sequence of characters that is seen as a unit by the compiler, like an identifier, number, an open/close symbol, etc.  This has been the standard for many years, it is implemented by tools like PMD/CPD and SonarQube. PMD is a framework for code analysis tools, CPD is its clone detection component, SonarQube is well known and used in many CI/CD lines.

In real life a code fragment may be duplicated and changed here and there. Because of the small changes this duplication may not be reported as the original sequence of duplicated tokens is broken into two smaller sequences, each smaller than 100 tokens. Therefore we propose to allow small 'gaps' in these clones, i.e., a few tokens that are not duplicated.

This new approach is now implemented in a tool called *Cupla*. It has been applied on various serious industrial software systems and open source repositories like Tensorflow, Spark and Linux. In allmost all cases we find significantly more duplication, about a factor 2.
The name *Cupla* is the Irish word for *couple*, as it tries to find code fragments that may not be identical clones but still be coupled.  Moreover, since the British have decided to quit the EU it does make sense to put more focus on the Irish language within the EU (:-).
This package is called ```cupla_java```, because it is a Java implementation of Cupla.

Below we go deeper into the concepts and the implemention of Cupla, the main thing is that it is reasonably fast and it has an acceptable memory consumption. 

- Analysing Tensorflow (4.575 C++-files, 1,3 million lines, 1 million lines of code) takes 26 seconds, whereas PMD/CPD takes 10 seconds. Cupla reports a code duplication of 13% (i.e., 13% of the lines of code are part of a duplicated fragment), PMD/CPD reports 5%.
- Analysing Linux (10.672 C-files, 6,3 million lines, 4,3 million lines of code) takes 1:30 min, whereas PMD/CPD takes 30 seconds. Cupla reports a code duplication of 7%, PMD/CPD reports 3%.
- Analysing PMD (1.769 Java-files, 175.000 lines, 99.000 lines of code) takes 4.5 seconds, whereas PMD/CPD takes 3,6 seconds. Cupla reports a code duplication of 13%, PMD/CPD reports only 1%[^1].

[^1]: This difference clearly stands out, some PMD examples are discussed below.

Cupla has been designed and implemented by Steven Klusener, a senior software engineer from Amsterdam. For many years he has been working in the software analysis of large software systems and software portfolios, as an engineer and consultant. After a while it became clear to him that there is more to say about code duplication than the output of CPD or similar tools. Cupla offers a first step forwards, by detecting more duplication and more meaningful clones.
In earlier prototypes Steven also worked on aggregating the detected clone information to more abstract levels, like which directories and packages within an entire portfolio are more or less copies of each other. Based on the duplication a similarity (a number between 0 and 1) was determined between software units (class files, packages, etc.). Using hierarchical clustering techniques these similarities were then visualized in hierarchical trees (also called dendograms). These techniques may be added to Cupla as well eventually.

## Some basic concepts

As said, in 'classic' clone analysis a *clone* is a number (> 1) of *clone instances*, each with a file, and a start and end position in that file that indicate the duplicated token sequence.  This token sequence has a certain minimal length (typically 100).  Normally layout tokens and comment tokens are ignored, so we deal only with the code tokens, like identifiers, operators, brackets, etc.  Since we are going to generalize the notion of a clone, we will refer to such a clone as a *pure clone* or a *match*.

In our Cupla approach we generalize this by allowing *gaps* that may not exceed a certain length (typically 30). A *gap* is a sequence of tokens within one of the clone instances that is *not* duplicated by all of the other clone instances. A *clone instance* is now an alternating sequence of a *match* (a *pure clone*, of length of 20 or more), followed by a *gap*, this alternating sequence is closed by one *match*. The resulting clones are called *pseudo clones*.

Here we give an example, for simplicitly we consider only sequences of integers (the spaces have no meaning, they are added to clarify the duplication between the two sequences):
```
1 : 1, 2, 3, 4, 5, 100, 101, 6, 7, 8, 9, 10, 11,      12, 13, 14, 15, 102
2 : 1, 2, 3, 4, 5,           6, 7, 8, 9, 10, 11, 200, 12, 13, 14, 15, 201
```

Assume that we use a *minimal match length* of 5, then a classic clone analysis would report two pure clones, namely ```1, 2, 3, 4, 5``` and ```6, 7, 8, 9, 10```. The last duplicated fragment, ```12, 13, 14, 15```, does not exceed the *minimal match length* threshold and is therefore discarded.

When we apply Cupla, with a *maximal gap length* of 3 and a *minimal match length* of 2, then we obtain a single pseudo clone with two clone instances. The first instance has a gap ```100, 101```, i. e., those two tokens are not duplicated within the other clone instance. This clone instance is denoted as follows:
```
[1, 2, 3, 4, 5], (100, 101), [6, 7, 8, 9, 10, 11, 12, 13, 14, 15]
```
where a match is indicated by ```[...]``` and a gap by ```(...)```.  The second clone instance can be given as:
```
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11], (200), [12, 13, 14, 15]
```
When the matches of each clone instance are concatenated we get a duplicated sequence of 15 tokens, so the token length of this pseudo clone is 15:
```
1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15
```

## Try it yourself

The directory ```jar``` contains a compiled version of this repo. You can run this as follows (if you have Java17 or higher)
```
    java -jar jars/cupla_java-<version>.jar <ini-file>
```
The argument ```<ini-file>``` should be a configuration file that at least should include an input directory.  If no output directory is given, then the output is given on stdout.  The extension is by default 'java', and the 'minTotalTokenLen' is by default 100.  Note that the parameters are case insensitive, so 'inputDir' equals 'inputdir'. So, such an ini-file may have the following contents:
```
    inputDir = your-input-directory
    extension = your-extension
    outputDir = your-output-directory
    minTotalTokenLen = 80
```
When you run Cupla a full list of all the settings is given, either on stdout (if you did not give an output directory) or in the file ```cupla.ini``` in the output directory. This file will show you all the settings from the given ini-file extended with all default values.

As input directory you can also have a Github repo. Cupla will then clone that repo into the output directory (which must be given). At least this works under Mac OS, and most likely also on other Unix-like platforms.

## Project organisation

(This section if not applicable for the repository ```cupla_jar```)

This application exports only the main package ```com.klusener.cupla```, it depends only on standard Java packages, see also the file ```src/main/java/module-info.java```.  The main package contains only one class file defining the ```main```-program that is triggered by the above Java-command.  The actual functionality is defined in the following subpackages, these are not exported:

- ```com.klusener.cupla.fileselector```: this subpackage defines a small class, called ```FileSelector``` that walks through the subdirectories of the input directory and selects the source files to be analysed, based on the given extension and some other settings.
- ```com.klusener.cupla.tokenize```: this subpackage defines the base class ```TokenizerStandard``` and its subclasses like ```TokenizerCpp```. These classes define the tokenization, i.e., how the source code is divided into a sequence of tokens (and how layout and comment are skipped). The base class defines definitions that are shared by various programming languages, like quoted strings, and the subclasses contain definitions that are specific for a particular language. For example, ```TokenizerCpp``` defines how *raw strings* should be recognized in C++. 
- ```com.klusener.cupla.clone```: this subpackage defines the duplication analysis, the core of Cupla. It contains the classes ```Gap```, ```CloneInstance```, ```Clone``` and ```CloneBuilder```. The last class contains the algorithms to build clones, with and without gaps. Finally, there is a class ```Reporter``` to generate the final report and a small summary.
- ```com.klusener.cupla.dto```: this subpackage defines all the shared data definitions, this subpackage is used by the first three subpackages.
- ```com.klusener.cupla.utils```: this subpackage defines a utility class and a simple logger, this subpackage is also used by the first three subpackages.


This repo is organized as a standard Maven-project.  All settings and dependencies are given in the ```pom.xml``` file.  The source code can be found in ```src/main/java```, the unit tests in ```src/test/java```.  There are 22 Java class files, with 4,447 lines of which 2,418 are lines of code.
There are 11 test files, with 1,439 lines of which 997 are lines of code.  The simple data classes have mainly getters and setters, for these classes no unit tests are available yet. There are also some end-to-end tests, the current test coverage (according to Eclipse) is 87%. The remaining 13% mainly involves exceptional cases, all parts with the major logic have been covered completely.

The project can be installed using maven:
```
    mvn install
```
Javadoc can be generated using
```
    mvn javadoc:javadoc
```
We have also added some configuration files for Eclipse, i.e. ```.classpath```, ```.project``` and ```.settings```.

Finally, for convenience we have added the generated jar-files in the folder ```jars```.

## End-to-End testing
(This section if not applicable for the repository ```cupla_jar```)

There are also some tests that take an number of input files run Cupla and compare all the results with a reference output.
These tests are defined in:
```
    src/test/java/com/klusener/cupla/TestCupla.java
```
This test files contains several tests, such as
```
	@Test
	void test_run_simple1() {
		run_test("run-simple1", Map.of(
				"inputDir", "simple1",
				"extension", "txt",
				"minTokenLen", "20"
				));
	}
```
This test will take all the file with extension ```.txt``` from the input directory:
```
    src/test/resources/cupla-tests-input/simple1
```
The output is dumped to:
```
    cupla-tests-output/run-simple1
```
And, if available, compared with:
```
    src/test/resources/cupla-tests-ref-output/run-simple1
```

## Some figures

The following table summarizes some of the results of Cupla and CPD.

| System      | Language | #Files | #Lines | #LOC | CPD% | Cupla% | Cuple time |
|:------------|:---------|:-------|:-------|:-----|:-----|:-------|:-----------| 
|Linux        | C        | 10.672 | 6,3 M  | 4,3 M| 3%   | 7%     | 1 min 30 sec|
|Tensorflow   | C++      | 4.575  | 1,3 M  | 1,0 M| 5%   | 13%    |  26 sec |
|Pytorch      | C++      | 1.481  | 0,6 M  | 0,5 M| 4%   | 10%    |  11 sec |
|Elasticsearch| Java     | 9.395  | 1,5 M  | 1,0 M| -    |  8%    |  18 sec |
|Spark        | C++      |   791  |  94 K  | 48 K | 5%   | 12%    |   3 sec |
|             | Scala    | 2.354  | 0,5 M  | 0,3 M| 3%   |  6%    |  50 sec |
|PMD          | Java     | 1.769  | 175 K  | 99 K | 1%   | 13%    |   5 sec |
|Cupla        | Java     |    22  |  4,5 K | 2,5 K| 0%   | 0%     |  0,4 sec|


## A brief comparison between Cupla and CPD

### A Tensorflow example
The following is based on a sample from Tensorflow (see ```src/test/resources/cupla-tests-input/tensorflow-sample2-*``` in case you have access to the Cupla repository).  CPD reports 11 (pure) clones, they are all covered by Cupla that in most cases has found larger (pseudo) clones.  For this sample CPD reports 4% duplication, Cupla 9%.

One example of a clone that is reported by Cupla and not by CPD is the following:
```
Found a clone of 17 lines (120 tokens) with 2 instances
  File compiler__mlir__xla__transforms__legalize_tf.cc, lines 1762-1778
  File compiler__mlir__xla__transforms__legalize_tf.cc, lines 1793-1810

  File compiler__mlir__xla__transforms__legalize_tf.cc, lines 1762-1778, match ratio: 0,85
    -----------------------------------------------------------
    :    Location loc = op.getLoc();
    |    Value features = op.features();
    |    auto featureType = features.getType();
    .
    .    // Use ConstantLike for `alpha` to match the shape of feature.
    |    auto alphaVal = chlo::getConstantLike(
    |        rewriter, loc, op.alpha().convertToFloat(), features);
    |    Value zeroVal = chlo::getConstantLike(rewriter, loc, 0.0, features);
    .
    :    Value leakyActivationVal = rewriter.create<mhlo::MulOp>(
    :        loc, features.getType(), features, alphaVal);
    .
    |    StringAttr compare_direction = StringAttr::get(rewriter.getContext(), "GT");
    |    Value compareGtZero = rewriter.create<mhlo::CompareOp>(
    |        loc, features, zeroVal, compare_direction);
    .
    |    rewriter.replaceOpWithNewOp<SelectOp>(op, featureType, compareGtZero,

  File compiler__mlir__xla__transforms__legalize_tf.cc, lines 1793-1810, match ratio: 0,85
    -----------------------------------------------------------
    :    Value gradients = op.gradients();
    |    Value features = op.features();
    |    auto featureType = features.getType();
    .
    .    // Use ConstantLike for `alpha` to match the shape of feature.
    |    auto alphaVal = chlo::getConstantLike(
    |        rewriter, loc, op.alpha().convertToFloat(), features);
    |    Value zeroVal = chlo::getConstantLike(rewriter, loc, 0.0, features);
    .
    :    Value leakyGradientVal = rewriter.create<mhlo::MulOp>(
    :        loc, features.getType(), gradients, alphaVal);
    .
    |    StringAttr compare_direction = StringAttr::get(rewriter.getContext(), "GT");
    .
    |    Value compareGtZero = rewriter.create<mhlo::CompareOp>(
    |        loc, features, zeroVal, compare_direction);
    .
    |    rewriter.replaceOpWithNewOp<SelectOp>(op, featureType, compareGtZero,
```
Every line is preceeded by a symbol, "|" indicates that the entire line is duplicated, ":" indicates that the line is partially duplicated and "." indicates a line that is skipped by Cupla.  We see that every instance has a number of gaps, such that they will not be reported by CPD. In the first we see a variable name ```leakyActivationVal``` and an argument ```features```, whereas in the second we have ```leakyGradientVal``` and ```gradients```. This duplication might be resolved by factoring out an auxiliary method.

The symbol "-" indicates that the line is part of a gap, and therefore it is not part of the clone. This case is not shown in the example above.

### A PMD example

The Java classes ```net.sourceforge.pmd.lang.modelica.rule.AbstractModelicaRule``` and ```net.sourceforge.pmd.lang.modelica.ast.ModelicaParserVisitorAdapter``` are copies of each other (the only difference is a number of imports in the first class). The duplicated fragment is a straight sequence of 3.590 tokens, almost 720 lines. This duplication is not reported by CPD, because the first class contains the tags ```CPD-OFF``` and ```CPD-ON``` to exclude the largest part of it.
When we add the following settings to our ini-file also Cupla will skip these fragments:
```
skipStartTag = CPD-OFF
skipEndTag = CPD-ON
```


Another example can be explained. There is a class ```net/sourceforge/pmd/lang/java/ast/JavaParserVisitorAdapter``` and the (deprecated) class ```net/sourceforge/pmd/lang/java/ast/JavaParserDecoratedVisitor.java```. This first one has methods like
```
    |    @Override
    |    public Object visit(ASTExtendsList node, Object data) {
    |        return visit((JavaNode) node, data);
    |    }
```
whereas the second one has an additional statement:
```
    |    @Override
    |    public Object visit(ASTExtendsList node, Object data) {
    -        visitor.visit(node, data);
    |        return visit((JavaNode) node, data);
    |    }
```
This additional statement is seen as a gap by Cupla (indicated by `-'), each pure clone is less than 100 tokens and therefore not reported by PMD/CPD.

If we add the following line to the ini-file, then all files that contain the string '@Deprecated' will be ignored:
```
   skipContentPattern = @Deprecated
```
After running Cupla again with this extra setting, 729 Java files have been selected with Cupla reports 5821 duplicated lines of code (12%) whereas PMD/CPD reports only 71 duplicated lines of code for these files (0%).

## The Settings

Below we briefly describe each setting that can be set in the ini-file.
Note that the names of the parameters are case insensitive, thus ```logLevel``` and ```loglevel``` are handled the same way.

### General Parameters
- ```runName``` (default ```cupla```), this name will be used in the output files, like ```cupla.log```, ```cupla-report.txt``` and ```cupla-summary.txt```.
- ```outputDir```, (default empty), if set then the given directory is used as output directory. The directory is created if it did not exist before. If it is not set, then the output is dumped to stdout.
- ```parallelize``` (default ```true```), if set, then the tokenization and the clone construction is spread over multiple parallel threads.
- ```loglevel``` (default ```warning```), other allowed values are ```severe```, ```info``` and ```debug```. The last value is mapped onto the the Java level ```FINE```.
- ```dumpDetails``` (default ```false```), if set, then some details are dumped. For ex. the output directory will have a subdirectory ```tokens``` that contains the tokens of each input file.
- ```runPmd```, (by default not set), if this parameter is definied it should contain the (full path to) the PMD-executable. If so, then PMD will be applied on the selected files (i.e., the files collected in the output file ```cupla-filelist.txt```). If Cupla is rerun the PMD-statistics will be included in ```cupla-summary.txt```.
- ```cpdReportFile```, (by default not set), if this parameter is defined it should contain the full path to a PMD/CPD-output file. If set, then the PMD-statistics will be included in ```cupla-summary.txt```. Note that either ```runPmd``` or ```cpdReportFile``` should be defined, but not both.
- ```defensive``` (default ```false```), if set, then a final sanity check is done on the constructed clones.

### File Selector Parameters
- ```inputDir```, (no default), this is the only mandatory parameter, it defines the input directory from where the input files are selected.
- ```extension```, (default ```java```), only files with the given extension are selected. Note that the extension also defines the 'language'-flavor of the tokenization. In case ```cpp``` or ```c```, then the CPP-tokenizer is used.
- ```skipFilePattern``` (default ```test```), all files or directories that match this pattern are skipped. This pattern can contain multiple subpatterns as usual, like ```test|drivers```.
- ```selectFilePattern``` (default undefined), if defined only files are selected if they match this pattern.

### Tokenization Parameters
- ```skipStartTag``` (default undefined), if set then all tokens following this tag until the ```skipEndTag``` are skipped by the clone construction.
- ```skipEndTag``` (default undefined), see ```skipStartTag```.
- ```baseNormalisation``` (default ```true```), if set, then the name of the file is replaced by ```__BASE__```. Suppose you have a file ```Abc_def.java```, then an identifier like ```myAbcDef``` is stored as ```my__BASE__```, idem for strings. This normalisation is particularly relevant for C++ implementation files in which all method declarations are qualified with the class name that normally corresponds with the file name. Without this normalisation (much) less duplication is found.
- ```caseInsensitive``` (default ```true```), if set, then all identifier and strings are stored in uppercase.
- ```skipPreprocessing``` (default ```true```, only relevant for C and C++), if set, then all lines that start with a preprocess keyword are skipped, in particular ```#include ...```.
- ```skipImports``` (default ```true```, not relevant for C and C++, mainly for Java), if set, then all import statements are skipped.
- ```qualifiedmaxlevel``` (default ```-1```), if set (i.e., if greater than 0) then all curly brackets until this depth are qualified.
So, if ```qualifiedMaxLevel=2```, then the top most opening/closing curly brackets will be stored as ```{.0``` and ```}.0```, and the next level as ```{.1``` and ```}.1```. All lower curly brackets will not be qualified. During clone construction such a qualified closing bracket, like ```}.0``` and ```}.1``` indicate the end of a clone, clone construction will not cross any of these symbols.  In case of C/C++ the namespaces will be added, so if a we have a 3-level namespace and qualifiedMaxLevel=2, then we will have ```{.0, ..., {.4```.  When C/C++ code contains preprocessing code with unmatching curly brackets, then this mechanism may lead to unwanted results, i.e., it will find less duplication as closing curly brackets are seen as the end of some top-most level.
Note that this mechanism is still experimental and still has to be evaluated thoroughly.


### Clone Construction Parameters
- ```minTotalTokenLen``` (default ```100```), clones with a total length less then this parameter are skipped. Note that the gaps of a clone do not count when the number of tokens of a clone instance are counted. Note also that the length of a clone is equal to the length of any of its clone instances.
- ```maxGapSplitCount``` (default ```1000```), if during the clone construction a split is reached with a large number of different tokens (i.e., larger than this parameter), then no gap construction is issued and for each token a subclone is created.
- ```maxGapLength``` (default 30), the maximum length of a gap.
- ```minmatchlength``` (default 20), the mimimum number of matching tokens that should follow a gap.
- ```checkNoStartTokens``` (default ```true```), if set then certain tokens (like the separatot symbol ```;```) are not allowed as the start of a clone.
- ```timeout``` (default ```3000```, in milliseconds), this timeout is used during the clone construction by each of the worker process, see the next parameter.
- ```timeoutminclonecount``` (default ```25```), each worker process has a list of clones, takes the first clone which may be split into subclones which are then added to the list of clones. Then moves on to the next clone, etcetera, until the list is empty.  If the timeout is reached and the number of clones exceeds this threshold, then the worker process returns its list of to-do clones to the main process and stops. The main process collects all remaining clones from its worker processes and redistributes them over a new set of worker processes. Without this timeout mechanism most worker process will terminate rather fast and a few worker process (those that got the 'complex' initial clones) will take a very long time, leading to a low level of parallelism.

## Future work

This is a first version, it still has to be tested on a larger scale.  Further potential improvements are given below.

### Revisiting the parameters
The detection of clones and their gaps is steered by a number of parameters:
- The *minimal total token length*, typically 100, the total number of tokens in a pseudo clone (gaps excluded);
- The *maximal gap length*, typically 30, the maximal number of tokens within a gap;
- the *minimal match length*, typically 20, the minimal number of tokens before or after a gap that should be duplicated within the other clone instances

Obviously, if different values are taken for these parameters then less of more duplication is detected. We have to apply Cupla on a larger scale, and discuss the results with various experts, before the *typical settings* are determined. Note that, following the minimal token length of CPD, it is common to apply a defensive policy, where false positives are avoided at the cost of false negatives.

On certain occasions we may deviate from this policy, for example in case the possibilities of refactoring a package are analysed, and all forms of duplication should covered. Then one may prefer to avoid false negatives at the cost of some false positives. In that case we may lower the minimal token length and the minimal pure clone length and increase the gap length. This will increase the number of detected clones significantly, although some of them will be related to boilerplate code instead of plain duplication. So, when to use which values for the parameters has to be further analysed.

### Revisiting the tokenizers
Currently we have one tokenizer, with some refinements for C++, that is applied to various languages.
So far we have tested Cupla on C++ and Java, and some Scala.
When more code is covered and more languages encountered, the requirements of one language may interfere with another language.
On the longer run a more serious implementation per language may have to be considered, for example using ANTLR.

## Licence
Cupla is designed and developed by Steven Klusener, who holds all rights on this software.
All persons and parties who have access to this repository may only read and use it for review purposes.


