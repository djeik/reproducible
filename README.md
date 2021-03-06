Reproducible
============

Reproducible is a script to assist scientists and others working on very
data-oriented projects. 

Motivation
----------

Frequently, we run a script to transform some input data into some output data,
but the output isn't quite to our liking. We run the script again with some
different parameters, possibly erasing the existing data. Now it looks a bit
better, so we try some more changes, each time squashing our old output. Now we
realize that a previous version was better, so we would like to go back, so we
must re-tweak our script's parameters again and again, hopefully getting
something that looks like what we had originally.

Now we're smarter: we decide to store each run in its own directory, named with
the date and time of the experiment. It's easy for us to go back and view past
iterations of the output, but suppose we find some past output that we really
like, but we wish that we could slightly tweak the parameters that generated
it. It's very difficult or tedious for us to recover those parameters,
especially if we just invoke the script on the command line. It's a little bit
easier if the invocation occurred in a script that is under version control,
but it's not impossible that we ran the script from a dirty reposity, i.e. one
in which we had uncommitted changes. Even if we're guaranteed that all our runs
come from clean repositories, then it is still quite tedious to search through
our git log to find the exact commit that generated a given output.

Reproducible solves all the above, getting rid of many of the headaches
associated with data-oriented programming. By ensuring that the scripts are not
dirty and by saving to the output folder the hash of the commit that generated
that output, it becomes trivial to recover the exact commit that generated that
output.

How it works
------------

Reproducible effectively creates a one-to-one mapping between commit hashes and
experimental output data. This guarantees that it is straightforward to roll
back to the exact working directory that generated a given set of output.

The user maintains a list of files under _reproducibility control_, and rather
than run the data processing scripts directly, we instead wrap the invocation
of the script in a call to `run_reproducible.py`. This wrapper
script will verify that the files listed under reproducibility control
have no uncommitted changes. If there are uncommitted changes, then the user is
simply invited to commit the work that has been done. It is possible to bypass
this check, but doing so severely limits reproducibility, as explained below. 

The wrapper script will then record in a file named `rev.txt` placed
in the experimental output folder the hash of the commit that generated this
experiment. This means that reverting to the code that generated this
experiment is as simple as opening `rev.txt` and running `git
checkout` on the hash contained there. From there it is possible to
create a new branch with changes effectively based on a past instance of the
experiment.

To help with looking up how the output got to be the way it is, Reproducible
will also save a file called `whatsnew.txt` alongside `rev.txt`, in which the
messages of the past few commits will be saved.

Setup
-----

Setup is very bad for now. Simply clone the repository somewhere, and copy
`run_reproducible.py` into your project folder. Due to the way
Python handles imports, this is necessary for relative imports to succeed. If
the inner script that is being wrapped does not perform any relative imports,
then there shouldn't be a problem, and the location of the wrapper shouldn't
matter much. 

The next step is to create a list of files to watch. Simply create a file named
`.reproducible` and list the files to watch, one path per line.

    dm_run.py
    dm_optimizer.py
    sa_tester.sh
    etc.

`run_reproducible.py` will fail if any of the following conditions
are not met:

* The directory in which `run_reproducible.py` is run must be
  (part of) a git repository.
* `.reproducible` must exist and all the files listed in it must
  exist.

Once the above has been carried out, you're ready to use Reproducible!

Usage
-----

`run_reproducible.py` is a generalized wrapper script. It's
invocation syntax is as follows:

    run_reproducible.py [OPTIONS...] <script> [ARGS...]

The possible options are the following:

* `-f | --force`: force the running of the script, even if one or
  more files from .reproducible are not committed.
* `-r | --reproducible`: (takes one argument) specify an alternate
  list of files to consider under reproducibility control. This setting can be
  used in scenarios where there are two or more subprojects in the same
  directory, and each should have its own separate list of
  reproducibility-controlled files. This setting will cause .reproducible to be
  ignore (in fact it can even be absent if -r is used instead.) 
* `-o | --output`: (takes one argument) set the directory where the
  reproducibility information should be stored. If this option is set, then the
  standard behaviour of using the last line of the inner script's standard
  output as the path to the output directory will be suppressed.
* `-n | --history`: (takes one argument) set the number of commit
  messages to show in the whatsnew.txt file saved alongside rev.txt. Setting
  this value to zero disables the creation of the whatsnew.txt.

The script passed to `run_reproducible.py` can be any executable. 
Whatever ARGS are specified on the command line are simply forwarded to the
inner script as-is. 

For `run_reproducible.py` to know where to store reproducibility
information, it must know what the experiment directory is. (It is expected that
the inner script create the experiment directory, generally based on the time
of the experiment, although this is not enforced.) The inner script's standard
output is captured, and its last line of standard output is taken to be the
directory where the reproducibility information should be stored. (This only
applies when not using the -r switch.) If this directory does not exist, then
`run_reproducible.py` will fail with exit code 1. The standard
output of the inner script is regardless forwarded to the terminal.

Due to the way the standard output is captured, i.e. via a pipe, the forwarding
will be odd in most applications that use the C standard library. It will
detect that output is being printed to a pipe rather than to a terminal, which
will cause the default buffering mode to be changed to block buffering as
opposed to the usual line buffering. It may therefore be necessary to manually
change the buffering mode of the inner application to use line buffering
regardless of the output sink.

The `-f` switch will make `run_reproducible.py` skip the
repository cleanliness check. This is not advisable, as it means that whatever
the inner script outputs cannot reliably be reproduced. Still, this is better
than just running the inner script directly, without wrapping it, since
`run_reproducible.py` will still record the SHA and will add to the
generated rev.txt the message "NOT CLEAN", to indicate that the repository's
watched files had local changes when the experiment was conducted. Furthermore,
`run_reproducible.py` will save the command-line used to invoke the
inner script to a file called `invocation.txt`. This information can
be useful when trying to reconstruct the data at a later time.

Since `run_reproducible.py` can take arguments from the command line
and forward them to `run_all`, it can be difficult to recover the
command-line that produced the experiment if the output of the experiment
hinges on what arguments were forwarded. Rather than force the user to write
additional wrapper scripts with the desired command-line arguments built-in,
Reproducible will simply save the command-line used to invoke it to a file name
`invocation.txt` in the same directory as `rev.txt`.

Rebasing
--------

Since Reproducible relies on commit hashes, using `git rebase` to
rewrite history is unadvisable as it will change the commit hashes. Our advice,
then, is simply to fork the repository should we wish to rebase.

Of course, if the commits in question did not produce any output, (e.g. a quick
succession of bugfixes,) then rebasing is preferred; each commit will therefore
contain a functioning instance of the project. If a commit produced broken
output, then it might be best to delete that output and combine the commit with
the subsequent ones, until the code is not broken. 

All these suggestions depend on the workflow, however, and should your team use
a different one, the only consideration is again that rewriting history can be
dangerous.

Pipelines
---------

Reproducible provides `run_reproducible.py` as its primary way of
creating easily reproducible data with minimal effort. For larger projects, in
which a more sophisticated (read: time-consuming) _pipeline_ is being used to
process data, rerunning the entire pipeline for every tweak is not feasible. 

A basic pipeline script called `run_reproducible_pipeline.py` is
provided for simple pipelines. A simple pipeline is one in which the output of
step n&gt;1 can depend only on the output(s) of step(s) m&lt;n and on
(unchanging) external files.

Naturally, as pipelines are more complex, the usage of
`run_reproducible_pipeline.py` is as well. Here is the
summary of its command line interface, and below, there is a concrete example
using it and its various switches.

    run_reproducible_pipeline.py [(-o|--output) <output directory>]
        [(-R|--results) <results directory>]  
        [-r <reproducible file>] [-p <.pipeline file>]
        ([--from <step>] [--to <step>] | [--only <step>])
        [--with <run>] [--ignore-missing-output]
        [(--continue | --everything)] [--force] [--final]

* `-o | --output`: specify the exact folder name where this run's
  output should be stored. 
  Default: generate the folder name with the current date and time.
* `-R | --results`: specify the path (relative to the current
  working directory) where the results directory is located.
  Default: `results` 
  If the directory does not exist, Reproducible will fail with an error
  message.
* `-r <reproducible file>`: specify the file that lists the files
  under reproducibility control (the commit-enforcing policy).
  Default: `.reproducible`
* `-p <pipeline file>`: specify the file that lists the steps in the
  pipeline and the names of the steps
  Default: `.pipeline`
* `--from N` `--to N` `--only N`: specify a
  range of steps to run.
  Default: all the steps listed in the pipeline file.
* `--with <run>`: specify the run from which to draw the output of
  previous steps. This switch has no effect if the entire pipeline is being
  run.
* `--ignore-missing-output`: when running from a step n>1, do not
  fail if output for steps m&lt;n does not exist.
* `--continue`: enforce that the next run should be a continuation
  of the previous one (or the one specified with `--with`). If
  proceeding as a continuation is not possible (e.g. for each step in the
  specified range, its output already exists in the previous run), fail with an
  error message.
* `--everything`: ignore previous runs, and run the entire pipeline
  from the start.
* `--force`: skip repository sanity check. This can result in
  irreproducible results.
* `--final`: save a hidden file named ".final" to the run directory.
  This will prevent Reproducible from considering this run as a previous run in 
  its automatic previous-run determination. Runs created with 
  `--force` are automatically made final.


### An example

Let's suppose we have the following directory structure:

    project/
    \   .gitignore  
        bin/
        \   run_all.sh
            step1.sh
            step2.sh
            step3.sh
        data/
        \   input_file.dat

Executing `run_all.sh` will create the following subdirectory of
`project`:

    results/
    \   step1/
        \   output
        step2/
        \   output
        step3/
        \   output

This would be the _normal_ way of doing things, with no reproducibility
control.

To use `run_reproducible_pipeline.py` we first need to write a
`.reproducible` file as described earlier, in order to enable to
commit-enforcing policy.  We also need to write a description of our pipeline,
which explains how each step should be performed.  We write this description in
a file named `.pipeline`:

    bin/step1.sh step1
    bin/step2.sh step2
    bin/step3.sh step3

`run_reproducible_pipeline.py` will effectively run
`run_reproducible.py` on what's in the first column of each line
each entry, but will enforce a certain amount of organization in the
`results` directory, and allow for a straightforward way of
recovering the hash of the commit that generated the output of any step in the
pipeline. 

Note: each of the paths in our `.pipeline` file above is prefixed
with `bin/` since we have chosen to place our pipeline 
specification file in the project's root folder. If we had placed it in the 
bin folder, this would not have been necessary. In other words, _the paths
in pipeline specification file are relative to the pipeline specification
file_.

The second column is reserved for the name of the step. This name is used as
the name of the directory in which the step's script is to write its output.
It is necessary for Reproducible to know the directory names for it to perform
previous step resolution as described below. These names are not allowed to
change, and they must be unique to a given project.

Each of the component scripts must accept as its first command-line argument
the folder where it should save its output to. By default,
`run_reproducible_pipeline.py` will make a folder named with the
current date and time. This directory's path is conveyed to each of the
component scripts, plus the name given in `.pipeline`. For example,
if I run a reproducible pipeline on 17 July 2014 at 3:32 PM, the first argument
given to step1.sh will be `2014:07:17 15:32:47/step1`, and that
is where `step1.sh` should write its output.

It is possible to provide an explicit folder to create with the `-o` 
(long form: `--output`) switch:

    run_reproducible_pipeline.py -o crazy_test

The command-line arguments of `run_reproducible_pipeline.py` are
saved to `invocation.txt` in the output directory, so it is possible
to recover this information (in case the folder gets renamed later, for
instance.)

The preferred way for a script to access the output of a previous step is to
simply use the same of that step, and a relative path:
`../step1/output` could be used from within step 2 to
access the output of step 1, which in turn should be step 2's input.
Furthermore, this means that it is straightforward to make a step that requires
the output of two or more previous steps, simply by referring to those steps by
name.

(Planned feature: simultaneous steps. Placing a `&` at the end of a
line in `.pipeline` would indicate to Reproducible that the
subsequent step may be performed at the same time as the current step. This
allows for a nice speedup in some cases, and allows for rudimentary branching 
pipelines.)

The component scripts in the pipeline do not have the requirement to have as
their last line of standard output the path to where the `rev.txt`
file should be saved. Reproducible already knows where it should go: it created
the experiment directory. 

(That's the main difference with non-pipelined execution: the callee is
expected to create the experiment directory and then inform Reproducible,
whereas in pipelined execution, Reproducible informs the callee of where it
should store its output.)

With `run_reproducible_pipeline.py`, our directory structure now
looks like this:

    project/
    \   .reproducible       (lists the scripts in bin/)
        .pipeline           (lists the scripts and output folder names)
        .gitignore
        bin/
            step1.sh
            step2.sh
            step3.sh
        data/
        \   input_file.dat
        results/
        \   2014:07:18 16:22:32
            \   step1/
                \   output
                step2/
                \   output
                step3/
                \   output
                rev.txt
                invocation.txt

N.B. In a real project, the output filenames will generally be more descriptive
than "output", i.e. we do so here only for the sake of example.

Let's suppose that the output of step1 is perfect: no more tweaking is required
for it. Step 2, however, still needs work, as we determine by looking at the
its output. We make some tweaks to step2.sh, (we commit our tweaks!) and then
want to rerun the pipeline. If we simply run
`run_reproducible_pipeline.py` again, then *everything* will be 
reconstructed, and that's no good. We need to specify that we would only
want to rebuild the chain of outputs from a given step. For this purpose, we
have the `--from` switch.

    run_reproducible_pipeline.py --from 2

The default behaviour for determining which step 1 output should be chosen is
to pick the step 1 that is the most recent (by file creation time stamp), since
the most recent version of the output is most likely the best one, but it's
possible to set a specific one by providing a path:

    run_reproducible_pipeline.py --from 2 --with "2014:07:18 16:22:32"

A path specified with `--with` is assumed to be relative to the
results directory.

In this case, the result of running either of the above two commands would
produce the same result:

    2014:07:18 19:36:29/
    \   step1 -> ../2014:07:18 16:22:32/step1
        step2/
        \   output
        step3/
        \   output
        rev.txt
        invocation.txt

It is also possible to supply the name of the step rather than its number.
There is a caveat, however: step names cannot consist solely of numbers. If a
step name is a number, then Reproducible will interpret that as the ordinal of
the step to run, rather than as its name. In other words, name resolution will
occur only if parsing to a number fails.

All steps prior to the one specified by the `--from` switch will be
generated as symlinks to the relevant run (either the most recent one or the
one specified with the `--with` switch). It it thus possible to
follow these symlinks to determine exactly which commit of the relevant driver
script(s) generated the output of the relevant step(s), by examining the
`rev.txt`.

As a counterpart to `--from`, we also provide `--to`.
Reproducible will run the pipeline up to and including the step identified by
the number given as an argument to the switch. The effect of `--from N
--to N` is therefore to run only step N. There is a shortcut switch to do
just that, namely `--only`.

Remarks:

 * if `--to` or `--only` are used such that the
   pipeline completes before running all its steps as listed in
   `.pipeline`, then symlinks to future steps will not be
   generated. By default, only symlinks to _previous_ steps are generated. To
   force the generation of symlinks to future steps, use
   `--link-future`.
 * if `--from` and/or `--to` is used alongside
   `--only`, then Reproducible will fail with an error message.
   Reproducible will also fail if the indices given are out of range, or if the
   value of `--to` is less than the value of `--from`

### Building a pipeline, piece by piece

`run_reproducible_pipeline.py` provides a convenience switch for
building up a pipeline. Suppose we start out by just writing
`step1.sh`, and we see the output is looking good. Now we write
`step2.sh`, but we can't simply run
`run_reproducible_pipeline.py`, since that would run step 1 again!

Or would it?

In fact, the default behaviour is to check the entries in
`.pipeline` and compare with the contents of the most recent run's
directory: if Reproducible determines that more steps have been added to
`.pipeline`, then it will infer a `--from` switch with
the appropriate value. In our example just above, the inferred value would be
two. If the number of steps is the same, then the entire pipeline will be run
again. Also, the entire pipeline will be run again if the number of steps has 
decreased, since there is currently no way for Reproducible to 
straightforwardly determine how to proceed.

Note that the use of the `--to` switch can influence the effect of
rebuilding/continuing inference. Suppose there are four steps in our pipeline
(we already have at least one run with the output of those four steps) and we
add three more steps, but use `--to 4`. The effect will be to run
all the four first steps over again, since it is the given _range_ of steps
that is checked for in the last run's output folder. Reproducible will see that
the specified steps exist, and therefore, the inferred behaviour will be to
rerun the pipeline entirely for those steps.

On the other hand, there is a case in which inference is not performed. Suppose
we have a pipeline with four steps, and we add three. There is at least one run
with the output of the first four, and we run with `--from 6`.
Reproducible will see that there is no output for step 5, and rather than infer
that it should run step five, it will fail with an error message. To
disambiguate, the user is required to either use `--from N` where
the output of step N-1 exists, or to use the
`--ignore-missing-output` switch. It is not advisable to use this
switch, however, since the resulting output directory will look rather strange
with missing steps in it.

### Explicit is better than implicit

Of course, relying on a program's ability to infer what the user wants can be
dangerous. As such, the following switches are provided to explicitly perform
the actions described in the previous section:

* `--everything`: run all the specified steps again.
* `--continue`: pick up where the pipeline left off, running the new
  steps based on the output of the previous steps.

Again, these command-line arguments will be saved to
`invocation.txt` in the run's folder.

Note that if `--continue` is used, but no new steps have been added,
Reproducible will simply fail with an error message.

Bugs and caveats
----------------

There are probably many more than those listed here. If you discover any, don't
hesitate to open an issue here or to submit a pull request.

 * Misbehaved buffers: the wrapper scripts effectively open a pipe to the inner
   scripts to collect their stdout. For the echoing of the inner script's
   stdout to stream correctly to the terminal, it might be necessary to disable
   output buffering in the inner script.
