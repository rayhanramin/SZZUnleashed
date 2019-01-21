# SZZUnleashed

A implementation of the SZZ algorithm as described by Śliwerski, Zimmermann, and Zeller in ["When Do
Changes Induce Fixes?"](https://www.st.cs.uni-saarland.de/papers/msr2005/), Proc. of the Int'l Workshop on Mining Software Repositories, Saint Louis, Missouri, USA, May 2005. The implementation also contains further improvements as described by Williams and Spacco in ["SZZ
Revisited: Verifying When Changes Induce Fixes"](https://www.researchgate.net/publication/220854597_SZZ_revisited_verifying_when_changes_induce_fixes).

## What is the usage of this algorithm?

The SZZ algorithm is used to find bug introducing commits from a set of bug
fixing commits. The bug introducing commits can be extracted either from a bug
tracking system such as JIRA or simply by searching for commits that states that
they are fixing something. The found bug introducing commits can then be used to
build datasets for machine learning purposes such as when buggy commits wants to
be found.

## Prerequisites:

* Java 8
* Gradle

## Usage SZZ algorithm

### Grab issues ###
The fetch script is an example of how one can extract issues from a bug tracking
system.
```python
python fetch.py
```
It creates a directory with issues. To convert these into a format where they can
be processed, use:
```python
python git_log_to_array.py <path_to_local_repo>
```
This creates a file `gitlog.json` that is used to link the issues to bug fixing
commits. Using the `find_bug_fixes.py` and this file, we can get a json file
that contains the Issue and its corresponding commit SHA-1, the commit date,
the creation date and the resolution date.

### Find the bug introducing commits ###

This implementation works regardless which language and file type. It uses
[JGIT](https://www.eclipse.org/jgit/) to parse a git repository.

To build a runnable jar file, use the gradle build script in the szz directory
like:

```shell
gradle build && gradle fatJar
```

Or if the algorithm should be runned without building a jar:

```shell
gradle build && gradle runJar
```

The algorithm tries to use as many cores as possible during runtime. The more
the merrier so to say.

To get the bug introducing commits from a repository using the file produced
by the previous issue to bug fix commit step, run:

```shell
java -jar szz_find_bug_introducers-<version_number>.jar -i <path_to_issues> -r <path_to_local_repo>
```
To assemble the results if the algorithm was able to use more than one core,
run the `assembler.py` script on the results directory.

## Output

The output can then be seen in three different files commits.json,
annotations.json and fix\_and\_bug\_introducing\_pairs.json.

The commits.json file includes all commits that have been blamed to be bug
introducing but which haven't been analyzed by any anything.

The annotations.json is a representation of the graph that is generated by the
algorithm in the blaming phase. Each bug fixing commit is linked to all possible
commits which could be responsible for the bug. Using the improvement from
Williams et al's, the graph also contains subgraphs which gives a deeper search
for responsible commits. It enables the algorithm to blame other commits than
just the one closest in history for a bug.

Lastly, the fix\_and\_bug\_introducing\_pairs.json includes all possible pairs
which could lead to a bug introduction and fix. This file is not sorted in any
way and it includes doublettes when it comes to both introducers and fixes. A
fix can be made several times and a introducer could be responsible for many
fixes.

## Feature Extraction ##
Now that the potential bug introducing commits has been identified, the
repository can be mined for features.

### Code Churns ###
The most simple features are the code churns. These are easily extracted by
just parsing each diff for each commit. The ones that are extracted are:

1. **Total lines of code** - Which simply is how many lines of code in total
for all changed files.
2. **Churned lines of code** - This is how many lines that have been inserted.
3. **Deleted lines of code** - The number of deleted lines.
4. **Number of Files** - The total number of changed files.

To get these features, run: `python assemble_code_churns.py <path_to_repo>
<branch>`

### Diffusion Features ###
The diffusion features are:

1. The number of modified subsystems.
2. The number of modified subdirectories.
3. The entropy of the change.

To extract the diffusion features, just run:
`python assemble_diffusion_features.py --repository <path_to_repo> --branch <branch>`

### Experience Features ###
Maybe the most uncomfortable feature group. The experience features are the
features that measures how much experience a developer has, both how recent
but also how much experience the developer has overall with the code.

The features are:

1. Overall experience.
2. Recent experience.

The script builds a graph to keep track of each authors experience. So the intial
run is:
`python assemble_experience_features.py --repository <repo_path> --branch <branch> --save-graph`

This will result in a graph which the script could use for future analysis

To rerun the analysis without generating a new graph, just run:
`python assemble_experience_features.py --repository <repo_path> --branch <branch>`

### History Features ###
The history are as follows:

1. The number of authors in a file.
2. The time between contributions made by the author.
3. The number of unique changes between the last commit.

The same as with the experience features, the script must initially generate a graph
where the file meta data is saved.
`python assemble_history_features.py --repository <repo_path> --branch <branch> --save-graph`

To rerun the script without generating a new graph, use:
`python assemble_history_features.py --repository <repo_path> --branch <branch>`

### Purpose Features ###
The purpose feature is just a single feature and that is if the commit is a fix o
not. To extract it use:

`python assemble_purpose_features.py --repository <repo_path> --branch <branch>`

### Coupling ###
A more complex number of features are the coupling features. These indicates
how strong the relation is between files and modules for a revision. This means
that two files can have a realtion even though they don't have a realtion
inside the source code itself. So by mining these, features that gives
indications in how many files that a commit actually has made changes to are
found.

The mining is made by a docker image containing the tool code-maat.

These features takes long time to extract but is mined using:

```python
python assemble_features.py --image code-maat --repo-dir <path_to_repo> --result-dir <path_to_write_result>
python assemble_coupling_features.py <path_to_repo>
```

It is also possible to specify which commits to analyze. This is done with the
CLI option `--commits <path_to_file_with_commits>`. The format of this file is
just lines where each line is equal to the corresponding commit SHA-1.

If the analyzation is made by several docker containers, one has to specify
the `--assemble` option which stands for assemble. This will collect and store
all results in a single directory.

The script is capable of checking if the are any commits that haven't been
analyzed. To do that, specify the `--missing-commits` option.

## Classification ##
Now that data has been assembled the training and testing of the ML model can
be made. To do this, simply run the model script in the model directory:
```python
python model.py train
```

## Authors

[Oscar Svensson](mailto:wgcp92@gmail.com)
[Kristian Berg](mailto:kristianberg.jobb@gmail.com)
