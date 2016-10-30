# run-google-java-format

Scripts to automatically download and run google-java-format,
and to slightly improve its output.

## run-google-java-format.py

The [google-java-format](https://github.com/google/google-java-format)
program reformats Java source code, but it creates poor formatting for
annotations in comments.  This script runs google-java-format and then
performs small changes to improve formatting of annotations in comments.
If called with no arguments, it reads from and writes to standard output.

This script reformats each file supplied on the command line according to
the Google Java style (by calling out to the google-java-format program,
https://github.com/google/google-java-format), but with improvements to
the formatting of annotations in comments.


## check-google-java-format.py</dt>

Given `.java` file names on the command line, reports any that
would be reformatted by the run-google-java-format.py program, and returns
non-zero status if there were any.
If called with no arguments, it reads from standard output.

This script checks whether the files supplied on the command line conform
to the Google Java style (as enforced by the google-java-format program,
but with improvements to the formatting of annotations in comments).
If any files would be affected by running run-google-java-format.py,
this script prints their names and returns a non-zero status.
If called with no arguments, it reads from standard input.
You could invoke this program, for example, in a git pre-commit hook.


## Integrating with a build system

Here are example targets you might put in a Makefile; integration with
other build systems is similar.

```
reformat:
	@wget -N https://raw.githubusercontent.com/plume-lib/run-google-java-format/master/run-google-java-format.py
	@run-google-java-format.py ${JAVA_FILES_FOR_FORMAT}

check-format:
	@wget -N https://raw.githubusercontent.com/plume-lib/run-google-java-format/master/check-google-java-format.py
	@check-google-java-format.py ${JAVA_FILES_FOR_FORMAT} || (echo "Try running:  make reformat" && false)
```

Here is an example of what you might put in a Git pre-commit hook:

```
CHANGED_JAVA_FILES=`git diff --staged --name-only --diff-filter=ACM | grep '\.java$'` || true
if [ ! -z "$CHANGED_JAVA_FILES" ]; then
    wget -N https://raw.githubusercontent.com/plume-lib/run-google-java-format/master/check-google-java-format.py
    python check-google-java-format.py ${CHANGED_JAVA_FILES}
fi
```

## Dealing with large changes when reformatting your codebase

When you first apply standard formatting, that may be disruptive to people
who have changes in their own branches/clones/forks.  (But, once you settle on consistent 
formatting, you will enjoy a number of benefits.
Applying standard formatting to your codebase makes it easier to read.  It
also simplifies use of a version control system:  it reduces the likelihood
of merge conflicts due to formatting changes, and it ensures that commits
and pull requests don't intermingle substantive changes with formatting
changes.)

Here are some notes about a possible way to deal with upstream
reformatting.  Comments/improvements are welcome.

For the person doing the reformatting:
 * Create a new branch and do your work there.
   ```git checkout -b reformat-gjf```
 * Tag the commit before the whitespace change as "before reformatting".
   ```git tag -a before-reformatting -m "Code before running google-java-format"```
 * Reformat by running a command such as:
    ```make reformat
    ant reformat
    gradle googleJavaFormat```
 * Examine the diffs to look for poor reformatting:
   ```git diff -w -b | grep -v '^[-+]import' | grep -v '^[-+]$'```
   or
   ```git diff -w -b | grep -v '^[-+]import' | grep -v '^[-+]$' | grep -v '@TADescription' | grep -v '@interface' | grep -v '@Target' | grep -v '@Default' | grep -v '@ImplicitFor' | grep -v '@SuppressWarnings' | grep -v '@SubtypeOf' | grep -v '@Override' | grep -v '@Pure' | grep -v '@Deterministic' | grep -v '@SideEffectFree'```
   A key example is a single statement that is the body of an if/for/while
   being moved onto the previous line with the boolean expression.
    * Search for occurrences of "^\+.*\) return ".
    * Search for occurrences of "^\+.*\(if\|while\|for\) (.*) [^{]".
    * Search for hunks that have fewer `+` than `-` lines.
   Add curly braces to get the body back on its own line.
   (You might want to do this in the master branch, and then start over with adding formatting.)
 * Run tests
 * Commit changes:
   ```git commit -m "Reformat code using google-java-format"```
 * Tag the commit that does the whitespace change as "after reformatting".
   ```git tag -a after-reformatting -m "Code after running google-java-format"```
 * Push both the commits and the tags:
   ```git push --tags```

For a client to merge the massive upstream changes:
Assuming before-reformatting is the last commit before reformatting
and after-reformatting is the reformatting commit:
 * Merge in the commit before the reformatting into your branch.
     ```git merge before-reformatting```
   Or, if you have "myremote" configured as remote, run these commands:
     ```git fetch myremote after-reformatting:after-reformatting
     git fetch myremote before-reformatting:before-reformatting```
 * Resolve any conflicts, run tests, and commit your changes.
 * Merge in the reformatting commit, preferring all your own changes.
     ```git merge after-reformatting -s recursive -X ours```
 * Run `ant reformat` or the equivalent command.
 * Commit any formatting changes.
 * Verify that this contains only changes you made (that is, the formatting
   changes were ignored):
     ```git diff after-reformatting...HEAD```
For a client of a client (such as a fork of a fork), the above instructions must be revised.
