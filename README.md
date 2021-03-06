# nlnz-m11n-tools-repository-splitter-updater National Library of New Zealand Modernisation Tools repository splitter and updater

Tools for for splitting a source repository into path-based repositories, creating patches for updating those split
repositories when the source repository changes and applying those patches.

## Synopsis

This repository contains gradle build tasks for splitting a source repository into path-based repositories,
creating patches for updating those split repositories when the source repository changes and applying those patches.

## Motivation

Many legacy repositories have grown large with the combination of many components. This often makes them difficult
to decouple. Splitting large repositories into smaller repositories based on specific functionality makes the work of
of decoupling monoliths much easier. There are other reasons why splitting a large repository makes sense as well.

## Requirements

This tool requires the following plugins:
 - `nz.govt.natlib.m11n.tools:automation-plugin`, which contains the necessary classes to do the splitting work
 - `nz.govt.natlib.m11n.tools:gradle-plugin`, which contains useful classes for gradle builds

## Conceptual model

### Source repository to destination repositories

Splitting takes place by taking a source repository and splitting it into one or more split-target repositories based on
specific file paths in the source repository. The parameters used to split the source repository are specified in a
JSON file. The structure of that file is described in `src/main/resources/sampleProjectParameters.json`.

### Patching once the repositories have been split

Once the source repository has been split, the split-target repositories can be updated by changes to the source
repository by creating patch files that are applied to the split-target repositories.

## Example operations

These examples go through the creation of the split-target repositories and the application of patches to them. They use
the tasks defined in this project to perform the repository splits operations.

## Recommended process

### First time

1. `cloneSourceRepository`.
2. See *Create initial base branch in source repository*.
3. `createSplitTargetRepositories` (first time, since these repositories haven't been created yet).
4. `extractAndPatchProjects`.
5. See *Push newly created repositories to github or other code management system*.
6. See *Create patch-merge branch and merge into master branch*.

### Subsequent updates

1. `cloneSourceRepository`.
2. See *Create appropriate to branch in the source repository*.
3. `cloneSplitTargetRepositories`.
4. See *Create appropriate branch in the split-target source repository*.
5. `extractAndPatchProjects`.
6. See *Push changes to github or other code management system*.
7. See *Create patch-merge branch and merge into master branch*.

### Create initial base branch in source repository

To make this process workable in the long-term, it's best to have a fixed base branch in the source branch to work from.
This branch would be based on the `master` branch at a certain point in time. The GUI provided by github or bitbucket
may not be able to specify the source branch, but since it defaults to `master` that may not be an issue. Otherwise,
we can perform the operation on the command line. The source repository must be cloned first and then perform the
following operation:
```
git checkout -b "<source-initial-patch-base-branch-name>" origin/master
git push origin HEAD
```

For purposes of documentation, we will refer to this initial patch branch as the *source-initial-patch-base*.

### Create appropriate to branch in the source repository

For subsequent patching operations, there needs to be a *to* branch for patching operations to occur. The patches are
created based on the differences between the *from* branch (which is the *source-initial-patch-base* branch) and the
*to* branch. For purposes of documentation we will refer to this subsequent patch branch as the *source-patch-point*
branch. This *to* or *source-patch-point* branch is created the same way as the *source-initial-patch-base* branch,
but this time it is based on the *master* branch at the commit point that we want to capture. Note that we could also
use a specific commit or tag as the commit point.

As before, the source repository must be cloned first. We then perform the following operation:
```
git checkout -b "<source-patch-point-branch-name>" origin/master
git push origin HEAD
```

### Create appropriate branch in the split-target source repository

Case 1:
If the split-target repository is newly created, we create a *split-target-patch-point* branch based off the *master*
branch:
```
git checkout -b "<split-target-patch-point-branch-name>" origin/master
git push origin HEAD
```

Case 2:
If the split-target source repository already exists, it would have a *split-target-patch-point* branch where the
initial patches have been applied. For subsequent patching operations, we want to create a branch based on that *last*
(or most recent) *split-target-patch-point* branch where we apply the new sets of patches. The GUI provided by github or
bitbucket may not be able to specify the source branch, so it may be necessary to perform the operation on the command
line. The split-target repositories must be cloned first and then in each of those cloned split-target repositories,
perform the following operation:
```
git checkout -b "<new-split-target-patch-point>" origin/<most-recent-split-target-patch-point>
git push origin HEAD
```

### Push changes to github or other code management system

Even if there are no changes to the patched branch, we still want this branch to exist in the target repository, as it
becomes the starting point for the next set of patch updates.

If the repository origin has been setup correctly, simply issue the command:
```
git push orgin HEAD
```

### Create patch-merge branch and merge into master branch

After the branch that contains the patches from the source repository (the new *split-target-patch-point* branch) has
been created and patched, this branch will need to be merged into the *master* (or development) branch. Even though the
branch may not be the master branch, for ease of documentation we'll refer to it as the *master* branch. We'll call the
branch that gets merged into the master branch the *patch-merge-branch*. It is *not* the *split-target-patch-point*
branch. The *patch-merge* branch is where the files that came from the source repository are moved and manipulated to
match the structure of the *master* branch. For example, you may need to change the directory structure or alter files
to fit the new project's direction and development plan. Eventually the *patch-merge branch* gets merged into the
*master* branch and the *patch-merge branch* gets deleted.

Because the *patch-merge* branch is based on the *master* branch, when the most recent *split-target-patch-point* branch
is merged into it, git will perform the same manipulations on the files that are being merged in. For example, a file is
moved to a new development structure and updated to reflect an underlying development need. That file movement is done
in commits in the *master* branch. Subsequent merges of a new *patch-merge* branch into the *master* branch will have
those changes applied to that file in its new location.

The only changes that will not occur are new files. New files from the source repository will need to be moved to
their new locations just like the other files that were moved there (most likely by using `git mv`). This means that
the *patch-merge* branch needs to be checked to make sure its structure is correct before it gets merged into the
*master* branch.

We want to keep the *split-target-patch-point* branch in the split-target repository, as these are the difference points
for subsequent updates. In other words, don't delete te *split-target-patch-point* branch! We first checkout the
*master* branch, do a fetch and pull to ensure that it is up to date. We then create a merge branch based on the
*master* branch*:
```
git checkout master
git fetch
git pull
git checkout --no-track -b "patch-branch-<name>-merge-into-master-branch" origin/master
git push origin HEAD
```

We then merge the patch branch into the merge branch that we have just created. The option `--no-edit` is to
automatically keep the generated commit message:
```
git merge origin/<most-recent-split-target-patch-point-branch-name> --no-edit
```

There may be merge conflicts. Resolve those conflicts. Move any necessary files to the correct location. Once the
conflicts have been resolved and committed, push up the merge branch:
```
git push origin HEAD
```

Then create a pull request to merge it into the *master* branch. The merge of the pull request should include the
deletion of the merge branch.

When using github to merge the pull request you may consider using either *Merge pull request* or *Rebase and merge* as
described by https://help.github.com/articles/about-pull-request-merges/. It depends on what kind of commit history
you want on the *master* branch.

If there are no changes between the *patch-merge* branch and the *master* branch (which means that there are no changes
between the most recent *split-target-patch-point* branch and the next most recent *split-target-patch-point* branch,
then there's no need to create a pull request (in fact, it can't be done because there is nothing to merge) and the
*patch-merge* branch can simply be deleted (as it is identical to the *master* branch). 


## Usage

### Common parameters

#### showCommandOutput

Show the output of commands on the console. This is useful for understanding what the tool is doing.

Example:
```
showCommandOutput=true
```

#### tempWorkFolder

`tempWorkFolder` is the folder where temporary files used in processing are created. Currently only the repositoryWorkflow
operations use this. This value must be set.

Example:
```
tempWorkFolder=/path/to/tmp/folder
```

#### reportsFolder

`reportsFolder` is the folder where reports from the repository conversion stored. Currently only the repositoryWorkflow
operations use this. This value must be set if reports are generated.

Example:
```
reportsFolder=/path/to/reports/folder
```

#### autocreateReportsFolder

`autocreateReportsFolder` indicates whether `reportsFolder` needs to be created. If this
is false and the `reportsFolder` does not exist and reports are run then an error will occur. The default is false.

Example:
```
autocreateReportsFolder=true
```

#### sourceRepositoryWorkingFolder

`sourceRepositoryWorkingFolder` is the root folder for copies of the source repository. The source repository goes through
several iterations to extract the necessary git history for the split-target repositories.

Example:
```
sourceRepositoryWorkingFolder="/go/split-work/source-repository-work"
```

#### autocreateSourceRepositoryWorkingFolder

`autocreateSourceRepositoryWorkingFolder` indicates whether sourceRepositoryWorkingFolder needs to be created. If this
is false and the `sourceRepositoryWorkingFolder` does not exist then an error will occur. The default is false.

Example:
```
autocreateSourceRepositoryWorkingFolder=true
```

#### sourceRepositoryNameBase

`sourceRepositoryNameBase` is the base name for the source code repository. The base name is used in combination
with the `repositoryWorkflowStartingSequenceIndex` to produce the starting repository, as in
"${sourceRepositoryNameBase}-${repositoryWorkflowStartingSequenceIndex}". This value must be set. This repository
is found in `sourceRepositoryWorkingFolder`.

Example:
```
sourceRepositoryNameBase="my-repository-name"
```

#### repositoryWorkflowStartingSequenceIndex

`repositoryWorkflowStartingSequenceIndex` is the starting sequence index used by RepositoryWorkflow when processing a
repository. The RepositoryWorkflow has the expectation that there is a repository with the name
"${sourceRepositoryNameBase}-${repositoryWorkflowStartingSequenceIndex}" and this is the starting repository for all
its configured operations. The default value is 0.

Example:
```
repositoryWorkflowStartingSequenceIndex=4
```

#### sourceCodeRepositoryServerUrl

The git repository URL for the originating code repository. This value must be set if the task `cloneSourceRepository`
is called.

Example:
```
sourceCodeRepositoryServerUrl="git@github.com:MyOrganization/my-repository-name.git'
```

#### sourceRepositoryGitBranch

The source repository will be checked out in the `sourceRepositoryGitBranch`. This value must be set.

Example:
```
sourceRepositoryGitBranch="my-split-branch-name"
```

#### preserveBranchNames

preserveBranchNames is the comma-separated list of branch names to preserve when extracting specific paths and files
from the source repository. These branch names are also checked out of the cloned repositories to ensure
they are visible if the remote origin is removed. The default is an empty list.

Example:
```
preserveBranchNames="branch-name-one,branch-name-two"
```

#### projectParametersJsonFile

The JSON file that contains the project parameters used for the various pipeline operations. An example project
parameters file is found in `src/main/resources/sampleProjectsParameters.json`. This file is required.

Example:
```
projectParametersJsonFile="/my/work/my-repository-splitting-project-parameters.json"
```

#### splitTargetProjectsRootFolder

`splitTargetProjectsRootFolder` is the root folder for the split-target repository projects.

Example:
```
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/targetRepositories"
```

#### autocreateSplitTargetProjectsRootFolder

`autocreateSplitTargetProjectsRootFolder` indicates whether the `splitTargetProjectsRootFolder` needs to be created.
If this is false and the `autocreateSplitTargetProjectsRootFolder` does not exist then an error will occur.
The default is false.

Example:
```
autocreateSplitTargetProjectsRootFolder=true
```

#### splitTargetRepositoryServerUrlBase

The git repository base URL for the split code repositories. This value must be set. The project name and '.git' is
appended to this for a specific code repository. The use of this variable assumes a consistent naming convention for
split-target repositories, in that their names all have a common prefix. A HTTPS cloning operation would use a URL
that starts with `https://` whereas a SSH cloning operation would use a URL that starts with an email address (such
as `git@github.com:` for github).

Note that the project name gets directly appended to the `splitTargetRepositoryServerUrlBase` without any intervening
characters. If there is a need for a intervening character, such as `/`, make sure that it gets included in the
`splitTargetRepositoryServerUrlBase`.

Example:
```
splitTargetRepositoryServerUrlBase="git@github.com:MyOrganization/my-split-target-repository-base-name-"
```

#### cloneSplitTargetRepositoriesRemoveRemoteOrigin

When cloning source repositories with the task `cloneSplitTargetRepositories`, this determines whether the remote origin is
removed (as in 'git remote rm origin') after the branches have been created and/or preserved. This is a safety measure
to ensure that changes to the cloned split-target repository cannot be pushed to the origin (if by chance the pipeline
operations cause some corruption or unwanted to state). The default is true.

Example:
```
cloneSplitTargetRepositoriesRemoveRemoteOrigin=true
```

#### patchTargetRepositoryFolder

patchTargetRepositoryFolder is the folder where the projects used to create patches can be found. This value must be set.

Example:
```
patchTargetRepositoryFolder="${splitTargetProjectsRootFolder}"
```

#### patchTargetRepositoryBranch

The patchTargetRepositoryBranch is the branch used as the end of the revision range for creating patches (as in the 'to'
argument in from..to).

Example:
```
patchTargetRepositoryBranch="my-changes-since-the-split-branch-name"
```

#### patchTargetRepositoryBaseBranch

The patchTargetRepositoryBaseBranch is the branch used as the base branch when creating the patchTargetRepositoryBranch,
as in 'git checkout --no-track -b ${patchTargetRepositoryBranch} origin/${patchTargetRepositoryBaseBranch}'.

Example:
```
patchTargetRepositoryBaseBranch="my-split-branch-name"
```

#### createPatchTargetRepositoryBranch

createPatchTargetRepositoryBranch indicates whether or not to create the patchTargetRepositoryBranch in the clone of
the source repositories. The default is false.

Example:
```
createPatchTargetRepositoryBranch=true
```

#### patchesFolderPath

patchesFolderPath is an override of the generated patches folder path used by the RepositoryWorkflow. Set this value
when applying specific patches from a specific folder, usually in conjunction with the task applyPatchesOnly and when
the includeProjectNames value is set to a single project. This value only makes sense when processing multiple projects.
By default this is not set (the value becomes null) and the patches folder is generated.

Example:
```
patchesFolderPath="/tmp/work/patches/patches-for-changes-from-my-split-branch-name-to-my-changes-since-the-split"
```

#### includeProjectNames

includeProjectNames is a comma-separated list of project names to include in the processing. includeProjectNames takes
precedence over excludeProjectNames (so if includeProjectNames is non-empty, then a project must be in the
includeProjectNames and not in the excludeProjectNames to be included in the list). May be empty.

Example:
```
includeProjectNames="split-project-one"
```

#### excludeProjectNames

excludeProjectNames is a comma-separated list of project names to exclude from the processing. includeProjectNames takes
precedence over excludeProjectNames (so if includeProjectNames is non-empty, then a project must be in the
includeProjectNames and not in the excludeProjectNames to be included in the list). May be empty.

Example:
```
excludeProjectNames="split-project-two"
```

### Common operations

Note that these common operations assume that the common parameters as listed above have already been set. The examples
do provide examples of setting the parameters. Operations can be combined and should work in proper order (for example
cloneSourceRepository will happen before cloneRepositories which happens before extractAndPatchProjects).

#### cloneSourceRepository

Clones the source repository into sourceRepositoryWorkingFolder. The repository name is
"${sourceRepositoryNameBase}-${repositoryWorkflowStartingSequenceIndex}". By default this would be
"${sourceRepositoryNameBase}-0".

Example:
```
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository"
autocreateSourceRepositoryWorkingFolder=true
sourceRepositoryGitBranch="my-source-git-branch"
preserveBranchNames="my-preserve-branch-name"
sourceCodeRepositoryServerUrl="git@github.com:MyOrganization/my-repository-name.git"
showCommandOutput="true"

gradle cloneSourceRepository \
 -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
 -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
 -PautocreateSourceRepositoryWorkingFolder=true \
 -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
 -PpreserveBranchNames="${preserveBranchNames}" \
 -PsourceCodeRepositoryServerUrl="${sourceCodeRepositoryServerUrl}" \
 -PshowCommandOutput="${showCommandOutput}" \
 --stacktrace --info
```

#### createSplitTargetRepositoriess (when the split-target repositories don't yet exist)

Creates the split-target repositories from the source repository based on the given parameteres.
This does all RepositoryWorkflow operations and produces a series of repositories.

*NOTE* that if these repositories already exist then it's better to use `cloneSplitTargetRepositories`.

Example:
```
projectParametersJsonFile="/my/project-parameters.json"
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository"
repositoryWorkflowStartingSequenceIndex="0"
sourceRepositoryGitBranch="my-split-origin"
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/split-target-repos"
autocreateSplitTargetProjectsRootFolder="true"
patchTargetRepositoryFolder="/my/repository-split-work/patch-repositories"
patchTargetRepositoryBranch="patch-branch-name"
patchTargetRepositoryBaseBranch="base-branch-name"
createPatchTargetRepositoryBranch=true
preserveBranchNames="base-branch-name"
includeProjectNames=
excludeProjectNames=
tempWorkFolder="/my/temp-folder"
reportsFolder="/my/reports-folder"
autocreateReportsFolder="true"
showCommandOutput="true"

gradle createSplitTargetRepositories \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder=true \
  -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
  -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
  -PrepositoryWorkflowStartingSequenceIndex="${repositoryWorkflowStartingSequenceIndex}" \
  -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PpatchTargetRepositoryFolder="${patchTargetRepositoryFolder}" \
  -PpatchTargetRepositoryBranch="${patchTargetRepositoryBranch}" \
  -PpatchTargetRepositoryBaseBranch="${patchTargetRepositoryBaseBranch}" \
  -PcreatePatchTargetRepositoryBranch="${createPatchTargetRepositoryBranch}" \
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PtempWorkFolder="${tempWorkFolder}" \
  -PreportsFolder="${reportsFolder}" \
  -PautocreateReportsFolder="${autocreateReportsFolder}" \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```

#### cloneSplitTargetRepositories (when the split-target repositories already exist)

Clones the split-target repositories into the given folder. The repository names are determined by the project names
found in the `projectParametersJsonFile` file in combination with the includeProjectNames and excludeProjectNames. Note
that the final branch name in preserveBranchNames is the one that is checked out after the repository is cloned.

*NOTE* that this only works if those repositories already exist. If the split-target repositories don't exist yet,
use `createSplitTargetRepositories`.

Example:
```
projectParametersJsonFile="/my/project-parameters.json"
splitTargetProjectsRootFolder="/my/split-repositories-work"
autocreateSplitTargetProjectsRootFolder="true"
preserveBranchNames="master,preserve-branch-one,preserve-branch-two"
cloneSplitTargetRepositoriesRemoveRemoteOrigin=false
includeProjectNames="my-include-project-names-one,my-include-project-names-two"
excludeProjectNames=
cloneSplitTargetRepositoriesRemoveRemoteOrigin="false"
splitTargetRepositoryServerUrlBase="git@github.com:MyOrganization/my-split-target-repository-base-name-"
showCommandOutput="true"

gradle cloneSplitTargetRepositories \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder="${autocreateSplitTargetProjectsRootFolder}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PcloneSplitTargetRepositoriesRemoveRemoteOrigin=${cloneSplitTargetRepositoriesRemoveRemoteOrigin} \
  -PsplitTargetRepositoryServerUrlBase=${splitTargetRepositoryServerUrlBase} \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```


#### extractAndPatchProjects

Extracts and patches a given set of projects. This does all RepositoryWorkflow operations.

Example:
```
projectParametersJsonFile="/my/project-parameters.json"
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository"
repositoryWorkflowStartingSequenceIndex="0"
sourceRepositoryGitBranch="my-split-origin"
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/split-target-repos"
autocreateSplitTargetProjectsRootFolder="true"
patchTargetRepositoryFolder="/my/repository-split-work/patch-repositories"
patchTargetRepositoryBranch="patch-branch-name"
patchTargetRepositoryBaseBranch="base-branch-name"
createPatchTargetRepositoryBranch=true
preserveBranchNames="base-branch-name"
includeProjectNames=
excludeProjectNames=
tempWorkFolder="/my/temp-folder"
reportsFolder="/my/reports-folder"
autocreateReportsFolder="true"
showCommandOutput="true"

gradle extractAndPatchProjects \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder=true \
  -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
  -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
  -PrepositoryWorkflowStartingSequenceIndex="${repositoryWorkflowStartingSequenceIndex}" \
  -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PpatchTargetRepositoryFolder="${patchTargetRepositoryFolder}" \
  -PpatchTargetRepositoryBranch="${patchTargetRepositoryBranch}" \
  -PpatchTargetRepositoryBaseBranch="${patchTargetRepositoryBaseBranch}" \
  -PcreatePatchTargetRepositoryBranch="${createPatchTargetRepositoryBranch}" \
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PtempWorkFolder="${tempWorkFolder}" \
  -PreportsFolder="${reportsFolder}" \
  -PautocreateReportsFolder="${autocreateReportsFolder}" \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```

#### createAndApplyPatchesOnly

Only creates and applies the patches. Does not do the source repository filter-branch and other operations.
Note that the `repositoryWorkflowStartingSequenceIndex` is 3 because that version has been filtered by the
previous steps.

```
projectParametersJsonFile="/my/project-parameters.json"
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository-name"
repositoryWorkflowStartingSequenceIndex="3"
sourceRepositoryGitBranch="my-split-origin"
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/split-target-repos"
autocreateSplitTargetProjectsRootFolder="true"
patchTargetRepositoryFolder="/my/repository-split-work/patch-repositories"
patchTargetRepositoryBranch="patch-branch-name"
patchTargetRepositoryBaseBranch="base-branch-name"
createPatchTargetRepositoryBranch=true
preserveBranchNames="base-branch-name"
includeProjectNames=
excludeProjectNames=
tempWorkFolder="/my/temp-folder"
reportsFolder="/my/reports-folder"
autocreateReportsFolder="true"
showCommandOutput="true"

gradle createAndApplyPatchesOnly \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder=true \
  -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
  -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
  -PrepositoryWorkflowStartingSequenceIndex="${repositoryWorkflowStartingSequenceIndex}" \
  -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PpatchTargetRepositoryFolder="${patchTargetRepositoryFolder}" \
  -PpatchTargetRepositoryBranch="${patchTargetRepositoryBranch}" \
  -PpatchTargetRepositoryBaseBranch="${patchTargetRepositoryBaseBranch}" \
  -PcreatePatchTargetRepositoryBranch="${createPatchTargetRepositoryBranch}" \
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PtempWorkFolder="${tempWorkFolder}" \
  -PreportsFolder="${reportsFolder}" \
  -PautocreateReportsFolder="${autocreateReportsFolder}" \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```

#### applyPatchesOnly

Only applies the patches. Does not do the source repository filter-branch and other operations. This operation would be
used when there has been some manual editing of the patch files and an attempt is made to apply the patches to
the project repository clone. These manually edited patched would be found in `patchesFolderPath`.
Note that the `repositoryWorkflowStartingSequenceIndex` is 3 because that version has been filtered by the
previous steps.

```
projectParametersJsonFile="/my/project-parameters.json"
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository-name"
repositoryWorkflowStartingSequenceIndex="3"
sourceRepositoryGitBranch="my-split-origin"
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/split-target-repos"
autocreateSplitTargetProjectsRootFolder="true"
patchTargetRepositoryFolder="/my/repository-split-work/patch-repositories"
patchTargetRepositoryBranch="patch-branch-name"
patchTargetRepositoryBaseBranch="base-branch-name"
patchesFolderPath="/path/to/my/special/edited/patches/folder"
createPatchTargetRepositoryBranch=true
preserveBranchNames="base-branch-name"
includeProjectNames=
excludeProjectNames=
tempWorkFolder="/my/temp-folder"
reportsFolder="/my/reports-folder"
autocreateReportsFolder="true"
showCommandOutput="true"

gradle applyPatchesOnly \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder=true \
  -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
  -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
  -PrepositoryWorkflowStartingSequenceIndex="${repositoryWorkflowStartingSequenceIndex}" \
  -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PpatchTargetRepositoryFolder="${patchTargetRepositoryFolder}" \
  -PpatchTargetRepositoryBranch="${patchTargetRepositoryBranch}" \
  -PpatchTargetRepositoryBaseBranch="${patchTargetRepositoryBaseBranch}" \
  -PpatchesFolderPath="${patchesFolderPath}" \
  -PcreatePatchTargetRepositoryBranch="${createPatchTargetRepositoryBranch}" \
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PtempWorkFolder="${tempWorkFolder}" \
  -PreportsFolder="${reportsFolder}" \
  -PautocreateReportsFolder="${autocreateReportsFolder}" \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```

#### applyPatchesOnly in combination with cloneRepositories

When trying to get a sequence of patches to work it's probably best to clone the target project repository and try
to apply the manually edited patches. This operation would be used when there has been some manual editing of the patch
files and an attempt is made to apply the patches to the project repository clone. These manually edited patched would
be found in 'patchesFolderPath'.
Note that the `repositoryWorkflowStartingSequenceIndex` is 3 because that version has been filtered by the
previous steps.

```
projectParametersJsonFile="/my/project-parameters.json"
sourceRepositoryWorkingFolder="/my/repository-split-work"
sourceRepositoryNameBase="my-source-repository-name"
repositoryWorkflowStartingSequenceIndex="3"
sourceRepositoryGitBranch="my-split-origin"
splitTargetProjectsRootFolder="${sourceRepositoryWorkingFolder}/split-target-repos"
autocreateSplitTargetProjectsRootFolder="true"
patchTargetRepositoryFolder="/my/repository-split-work/patch-repositories"
patchTargetRepositoryBranch="patch-branch-name"
patchTargetRepositoryBaseBranch="base-branch-name"
patchesFolderPath="/path/to/my/special/edited/patches/folder"
createPatchTargetRepositoryBranch=true
preserveBranchNames="base-branch-name"
cloneRepositoriesRemoveRemoteOrigin=true
includeProjectNames=
excludeProjectNames=
tempWorkFolder="/my/temp-folder"
reportsFolder="/my/reports-folder"
autocreateReportsFolder="true"
showCommandOutput="true"

gradle cloneRepositories applyPatchesOnly \
  -PprojectParametersJsonFile="${projectParametersJsonFile}" \
  -PsplitTargetProjectsRootFolder="${splitTargetProjectsRootFolder}" \
  -PautocreateSplitTargetProjectsRootFolder=true \
  -PsourceRepositoryWorkingFolder="${sourceRepositoryWorkingFolder}" \
  -PsourceRepositoryNameBase="${sourceRepositoryNameBase}" \
  -PrepositoryWorkflowStartingSequenceIndex="${repositoryWorkflowStartingSequenceIndex}" \
  -PsourceRepositoryGitBranch="${sourceRepositoryGitBranch}" \
  -PpreserveBranchNames="${preserveBranchNames}" \
  -PpatchTargetRepositoryFolder="${patchTargetRepositoryFolder}" \
  -PpatchTargetRepositoryBranch="${patchTargetRepositoryBranch}" \
  -PpatchTargetRepositoryBaseBranch="${patchTargetRepositoryBaseBranch}" \
  -PpatchesFolderPath="${patchesFolderPath}" \
  -PcreatePatchTargetRepositoryBranch="${createPatchTargetRepositoryBranch}" \
  -PcloneRepositoriesRemoveRemoteOrigin="${cloneRepositoriesRemoveRemoteOrigin}"
  -PincludeProjectNames="${includeProjectNames}" \
  -PexcludeProjectNames="${excludeProjectNames}" \
  -PtempWorkFolder="${tempWorkFolder}" \
  -PreportsFolder="${reportsFolder}" \
  -PautocreateReportsFolder="${autocreateReportsFolder}" \
  -PshowCommandOutput="${showCommandOutput}" \
  --stacktrace --info
```

#### restoreRemoteOriginForProjects

Restores the remote origin for the filtered project list.

#### pushBranchOriginForProjects

Pushes the branch to origin.

#### createMergeBranchForProjects

Creates a merge branch for the filtered project list.

## Pushing a split-target repository to github

After creating a split-target repository with the task `createSplitTargetRepositories`, you may want to create this
repository in github and push your changes to github. There are several steps in this process.

For the description of this process, we will use the example split-target repository of `example-split-target-repository`
which has a `patchTargetRepositoryBaseBranch` of `split-target-base-branch`.

We assume that as part of the split-target repository creation that a `master` branch has been created along with the
`patchTargetRepositoryBaseBranch` (or some other branch that reflects the default branch for the repository). It's useful
to confirm that this `master` branch has the same history (found by using `git log` as the `patchTargetRepositoryBaseBranch`).
You can verify the branches that exist in the split-target repository by using `git branch --all`.

### Create the repository in github

Create the split-target repository in github using its web interface. Create this repository with the following
parameters:
- It does not have a description
- It is created _without_ README
- It has a license of _None_
- It has a .gitignore of _None_

### Add the repository remote corresponding to the github repository

Example:
```
git remote add origin git@github.com:<organisation-name>/<repository-name>.git
```

### Push the split-target branches to github

Example:
```
git checkout master
git push origin HEAD
git checkout split-target-base-branch
git push origin HEAD
```

### Verify that the default branch for your github repository is the `master` branch

Go to the github repository settings to confirm that the default branch is the `master` branch.

### Clone the github repository to verify that it has been created as expected

## Versioning

This pipeline does not produce any artifacts.

## Contributors

See git commits to see who contributors are. Issues are tracked through the github issue tracking.

## License

&copy; 2018 &mdash; 2019 National Library of New Zealand. MIT License. All rights reserved.
