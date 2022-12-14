# Cupla: An improved code duplication analysis 

## Introduction

Since several decades software quality has been an ongoing issue, it has been generally understood that *technical debt* (i.e., a low level of software quality) is a liability for an organisation leading to higher and less predictable maintenance costs. Nowadays it is quite standard to adopt tooling within the sofware delivery process (such as CI/CD) that applies a number of checks on the source code.

One of the most important checks is the detection of duplicated code, to avoid/reduce the following:
- Duplicated code also means duplicated maintenance;
- If duplication is not known (which is the case most of the time), and a bug is resolved in one place, the bug will still remain at the other place, leading to negative surprises over time.
Duplication can be avoided/resolved by applying software engineering techniques like object orientation. A small amount of duplication is acceptable, it is generally agreed that the duplication ratio should not exceed 5%.

Classically duplicated code fragments, called *clones*, are reported when they have a length of 100 tokens or more. A *token* is a sequence of characters that is seen as a unit by the compiler, like an identifier, number, some open/close symbol, etc.  This has been the standard for many years, it is implemented by tools like PMD/CPD and SonarQube. PMD is a framework for code analysis tools, CPD is its clone detection component, SonarQube is well known and used in many CI/CD lines.

In real life a code fragment may be duplicated and changed here and there. Because of the small changes this duplication may not be reported as the original sequence of duplicated tokens is broken into two smaller sequences, each smaller than 100 tokens. Therefore we propose to allow small 'gaps' in these clones, i.e., a few tokens that are not duplicated.

This new approach is now implemented in a tool called *Cupla*. It has been applied on various serious industrial software systems and open source repositories like Tensorflow, Spark and Linux. In allmost all cases we find significantly more duplication, about a factor 2.
The name *Cupla* is the Irish word for *couple*, as it tries to find code fragments that may not be identical clones but still be coupled.  Moreover, since the British have decided to quit the EU it does make sense to put more focus on the Irish language within the EU (:-).
This package is called ```cupla_java```, because it is a Java implementation of Cupla.

Below we go deeper into the concepts and the implemention of Cupla, the main thing is that it is reasonably fast and it has an acceptable memory consumption. 

- Analysing Tensorflow (4575 files, 1.4 million lines, almost 1 million lines of code) takes 25 seconds, whereas PMD/CPD takes 10 seconds. Cupla reports a code duplication of 13% (i.e., 13% of the lines of code are part of a duplicated fragment), PMD/CPD reports 5%.
- Analysing Linux (10672 files, 6.3 million lines, 4.3 million lines of code) takes 1:24 min, whereas PMD/CPD takes 30 seconds. Cupla reports a code duplication of 7%, PMD/CPD reports 3%.

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
    java -jar cupla_java-<version>.jar <ini-file>
```
The argument ```<ini-file>``` is a configuration file that should at least include an input directory.  If no output directory is given, then the output is given on stdout.  The extension is by default 'java', and the 'minTotalTokenLen' is by default 100.  Note that the parameters are case insensitive, so 'inputDir' equals 'inputdir'. So, such an ini-file may have the following contents:
```
    inputDir = your-input-directory
    extension = your-extension
    outputDir = your-output-directory
    minTotalTokenLen = 80
```
When you run Cupla a full list of all the settings is given, either on stdout (if you did not give an output directory) or in the file ```cupla.ini``` in the output directory. This file will show you all the settings from the given ini-file extended with all default values.

As input directory you can also have a Github repo. Cupla will then clone that repo into the output directory (which must be given). At least this works under Mac OS, and most likely also on other Unix-like platforms.


## A brief comparison between Cupla and CPD
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

## Future work

This is a first version, it still has to be tested on a larger scale.  Further potential improvements are given below.

### Revisiting the parameters
The detection of clones and their gaps is steered by a number of parameters:
- The *minimal total token length*, typically 100, the total number of tokens in a pseudo clone (gaps excluded);
- The *maximal gap length*, typically 30, the maximal number of tokens within a gap;
- the *minimal match length*, typically 20, the minimal number of tokens before or after a gap that should be duplicated within the other clone instances

Obviously, if different values are taken for these parameters then less of more duplication is detected. We have to apply Cupla on a larger scale, and discuss the results with various experts, before the *typical settings* are determined. Note that, following the minimal token length of CPD, it is common to apply a defensive policy, where false positives are avoided at the cost of false negatives.

On certain occasions we may deviate from this policy, for example in case the possibilities of refactoring a package are analysed, and all forms of duplication should covered. Then one may prefer to avoid false negatives at the cost of some false positives. In that case we may lower the minimal token length and the minimal pure clone length and increase the gap length. This will increase the number of detected clones significantly, although some of them will be related to boilerplate code instead of plain duplication. So, when to use which values for the parameters has to be further analysed.

### Clones that follow the structure
We would like to see only clones that are consistent with the structure of the code. For example, we would like to avoid a clone that starts with the end of one method, followed by the start of the next method. 

In the above example we see a clone that starts with ```...();```, probably it is more intuitive to drop this line entirely from the clone.

### Revisiting the tokenizers
Currently we have one tokenizer, with some refinements for C++, that is applied to various languages.
So far we have tested Cupla on C++ and Java, and some Scala.
When more code is covered and more languages encountered, the requirements of one language may interfere with another language.
On the longer run a more serious implementation per language may have to be considered, for example using ANTLR.

## Licence
Cupla is designed and developed by Steven Klusener, who holds all rights on this software.
All persons and parties who have access to this repository may only read and use it for review purposes.


