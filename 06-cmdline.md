---
layout: lesson
root: ../..
---



## Command-Line Programs

The R Console and other interactive tools like RStudio are great for prototyping code and exploring data, but sooner or later we will want to use our program in a pipeline or run it in a shell script to process thousands of data files.
In order to do that, we need to make our programs work like other Unix command-line tools.
For example, we may want a program that reads a data set and prints the average inflammation per patient:

~~~
$ Rscript readings.R --mean inflammation-01.csv
5.45
5.425
6.1
...
6.4
7.05
5.9
~~~

but we might also want to look at the minimum of the first four lines

~~~
$ head -4 inflammation-01.csv | Rscript readings.R --min
~~~

or the maximum inflammations in several files one after another:

~~~
$ Rscript readings.R --max inflammation-*.csv
~~~

Our overall requirements are:

1. If no filename is given on the command line, read data from [standard input](../../gloss.html#standard-input).
2. If one or more filenames are given, read data from them and report statistics for each file separately.
3. Use the `--min`, `--mean`, or `--max` flag to determine what statistic to print.

To make this work, we need to know how to handle command-line arguments in a program, and how to get at standard input.
We'll tackle these questions in turn below.

<div class="objectives" markdown="1">
#### Objectives

*   Use the values of command-line arguments in a program.
*   Handle flags and files separately in a command-line program.
*   Read data from standard input in a program so that it can be used in a pipeline.
</div>

### Command-Line Arguments

Using the text editor of your choice, save the following line of code in a text file called `session-info.R`:


<div class='out'><pre class='out'><code>sessionInfo()
</code></pre></div>

The function, `sessionInfo`, outputs the version of R you are running as well as the type of computer you are using (as well as the versions of the packages that have been loaded).
This is very useful information to include when asking others for help with your R code.

Now we can run the code in the file we created from the Unix Shell using `Rscript`:


<pre class='in'><code>Rscript session-info.R</code></pre>




<div class='out'><pre class='out'><code>R version 3.1.1 (2014-07-10)
Platform: x86_64-pc-linux-gnu (64-bit)

locale:
 [1] LC_CTYPE=en_US.UTF-8       LC_NUMERIC=C              
 [3] LC_TIME=en_US.UTF-8        LC_COLLATE=en_US.UTF-8    
 [5] LC_MONETARY=en_US.UTF-8    LC_MESSAGES=en_US.UTF-8   
 [7] LC_PAPER=en_US.UTF-8       LC_NAME=C                 
 [9] LC_ADDRESS=C               LC_TELEPHONE=C            
[11] LC_MEASUREMENT=en_US.UTF-8 LC_IDENTIFICATION=C       

attached base packages:
[1] stats     graphics  grDevices utils     datasets  base     
</code></pre></div>

> **Tip:** If that did not work, remember that you must be in the correct directory.
You can determine which directory you are currently in using `pwd` and change to a different directory using `cd`.
For a review, see this [lesson](../shell/01-filedir.html) or the [Unix Shell Reference](../ref/01-shell.html).

Now let's create another script that does something more interesting. Write the following lines in a file named `print-args.R`:


<div class='out'><pre class='out'><code>args <- commandArgs()
args
</code></pre></div>

The function `commandArgs` extracts all the command line arguments and returns them as a vector.
Let's see what happens when we run this program in the Unix Shell:


<pre class='in'><code>Rscript print-args.R</code></pre>




<div class='out'><pre class='out'><code>[1] "/usr/lib/R/bin/exec/R" "--slave"               "--no-restore"         
[4] "--file=print-args.R"   "--args"               
</code></pre></div>

From this output, we learn that `Rscript` is just a convenience command for running R scripts.
The first argument in the vector is the path to the `R` executable.
The following are all command-line arguments that affect the behavior of R.
From the R help file:

*  `--slave`: Make R run as quietly as possible
*  `--no-restore`:  Don't restore anything that was created during the R session
*  `--file`: Run this file
*  `--args`: Pass these arguments to the file being run

Thus running a file with Rscript is an easier way to run the following:


<pre class='in'><code>R --slave --no-restore --file=print-args.R --args</code></pre>




<div class='out'><pre class='out'><code>[1] "/usr/lib/R/bin/exec/R" "--slave"               "--no-restore"         
[4] "--file=print-args.R"   "--args"               
</code></pre></div>

If we run it with a few arguments, however:


<pre class='in'><code>Rscript print-args.R first second third</code></pre>




<div class='out'><pre class='out'><code>[1] "/usr/lib/R/bin/exec/R" "--slave"               "--no-restore"         
[4] "--file=print-args.R"   "--args"                "first"                
[7] "second"                "third"                
</code></pre></div>

then `commandArgs` adds each of those arguments to the vector it returns.
Since the first elements of the vector are always the same, we can tell `commandArgs` to only return the arguments that come after `--args`.
Let's update `print-args.R` and save it as `print-args-trailing.R`:


<div class='out'><pre class='out'><code>args <- commandArgs(trailingOnly = TRUE)
args
</code></pre></div>

And then run `print-args-trailing` from the Unix Shell:


<pre class='in'><code>Rscript print-args-trailing.R first second third</code></pre>




<div class='out'><pre class='out'><code>[1] "first"  "second" "third" 
</code></pre></div>

Now `commandArgs` returns only the arguments that we listed after `print-args-trailing.R`.

With this in hand, let's build a version of `readings.R` that always prints the per-patient (per-row) mean of a single data file.
The first step is to write a function that outlines our implementation, and a placeholder for the function that does the actual work.
By convention this function is usually called `main`, though we can call it whatever we want.
Write the following code in a file called `readings-01.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  filename <- args[1]
  dat <- read.csv(file = filename, header = FALSE)
  mean_per_patient <- apply(dat, 1, mean)
  return(mean_per_patient)
}
</code></pre></div>


This function gets the name of the file to process from the first element returned by `commandArgs`.
Here's a simple test to run from the Unix Shell:


<pre class='in'><code>Rscript readings-01.R inflammation-01.csv</code></pre>

There is no output because we have defined a function, but haven't actually called it.
Let's add a call to `main` and save it as `readings-02.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  filename <- args[1]
  dat <- read.csv(file = filename, header = FALSE)
  mean_per_patient <- apply(dat, 1, mean)
  return(mean_per_patient)
}

main()
</code></pre></div>


<pre class='in'><code>Rscript readings-02.R inflammation-01.csv</code></pre>




<div class='out'><pre class='out'><code> [1] 5.450 5.425 6.100 5.900 5.550 6.225 5.975 6.650 6.625 6.525 6.775 5.800
[13] 6.225 5.750 5.225 6.300 6.550 5.700 5.850 6.550 5.775 5.825 6.175 6.100
[25] 5.800 6.425 6.050 6.025 6.175 6.550 6.175 6.350 6.725 6.125 7.075 5.725
[37] 5.925 6.150 6.075 5.750 5.975 5.725 6.300 5.900 6.750 5.925 7.225 6.150
[49] 5.950 6.275 5.700 6.100 6.825 5.975 6.725 5.700 6.250 6.400 7.050 5.900
</code></pre></div>

#### Challenges

  + Write a command-line program that does addition and subtraction:


<pre class='in'><code>Rscript arith.R 1 + 2</code></pre>




<div class='out'><pre class='out'><code>[1] 3
</code></pre></div>


<pre class='in'><code>Rscript arith.R 3 - 4</code></pre>




<div class='out'><pre class='out'><code>[1] -1
</code></pre></div>



  + What goes wrong if you try to add multiplication using `*` to the program?
  


  + Using the function `list.files` introduced in a previous [lesson](03-loops-R.html), write a command-line program that lists all the files in the current directory that contain a specific pattern:


<pre class='in'><code>Rscript find-pattern.R inflammation</code></pre>




<div class='out'><pre class='out'><code> [1] "inflammation-01.csv" "inflammation-02.csv" "inflammation-03.csv"
 [4] "inflammation-04.csv" "inflammation-05.csv" "inflammation-06.csv"
 [7] "inflammation-07.csv" "inflammation-08.csv" "inflammation-09.csv"
[10] "inflammation-10.csv" "inflammation-11.csv" "inflammation-12.csv"
</code></pre></div>



### Handling Multiple Files

The next step is to teach our program how to handle multiple files.
Since 60 lines of output per file is a lot to page through, we'll start by using three smaller files, each of which has three days of data for two patients.
Let's investigate them from the Unix Shell:


<pre class='in'><code>ls small-*.csv</code></pre>




<div class='out'><pre class='out'><code>small-01.csv
small-02.csv
small-03.csv
</code></pre></div>


<pre class='in'><code>cat small-01.csv</code></pre>




<div class='out'><pre class='out'><code>0,0,1
0,1,2
</code></pre></div>


<pre class='in'><code>Rscript readings-02.R small-01.csv</code></pre>




<div class='out'><pre class='out'><code>[1] 0.3333333 1.0000000
</code></pre></div>

Using small data files as input also allows us to check our results more easily: here, for example, we can see that our program is calculating the mean correctly for each line, whereas we were really taking it on faith before.
This is yet another rule of programming: "[test the simple things first](../../rules.html#test-simple-first)".

We want our program to process each file separately, so we need a loop that executes once for each filename.
If we specify the files on the command line, the filenames will be returned by `commandArgs(trailingOnly = TRUE)`.
We'll need to handle an unknown number of filenames, since our program could be run for any number of files.

The solution is to loop over the vector returned by `commandArgs(trailingOnly = TRUE)`.
Here's our changed program, which we'll save as `readings-03.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  for (filename in args) {
    dat <- read.csv(file = filename, header = FALSE)
    mean_per_patient <- apply(dat, 1, mean)
    print(mean_per_patient)
  }
}

main()
</code></pre></div>

and here it is in action:


<pre class='in'><code>Rscript readings-03.R small-01.csv small-02.csv</code></pre>




<div class='out'><pre class='out'><code>[1] 0.3333333 1.0000000
[1] 13.66667 11.00000
</code></pre></div>

**Note**: at this point, we have created three versions of our script called `readings-01.R`, `readings-02.R`, and `readings-03.R`.
We wouldn't do this in real life: instead, we would have one file called `readings.R` that we committed to version control every time we got an enhancement working.
For teaching, though, we need all the successive versions side by side.

#### Challenges

  + Write a program called `check.R` that takes the names of one or more inflammation data files as arguments and checks that all the files have the same number of rows and columns.
  What is the best way to test your program?





### Handling Command-Line Flags

The next step is to teach our program to pay attention to the `--min`, `--mean`, and `--max` flags.
These always appear before the names of the files, so let's save the following in `readings-04.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  action <- args[1]
  filenames <- args[-1]
  
  for (f in filenames) {
    dat <- read.csv(file = f, header = FALSE)
    
    if (action == "--min") {
      values <- apply(dat, 1, min)
    } else if (action == "--mean") {
      values <- apply(dat, 1, mean)
    } else if (action == "--max") {
      values <- apply(dat, 1, max)
    }
    print(values)
  }
}

main()
</code></pre></div>

And we can confirm this works by running it from the Unix Shell:


<pre class='in'><code>Rscript readings-04.R --max small-01.csv</code></pre>




<div class='out'><pre class='out'><code>[1] 1 2
</code></pre></div>

but there are several things wrong with it:

1.  `main` is too large to read comfortably.

2.  If `action` isn't one of the three recognized flags, the program loads each file but does nothing with it (because none of the branches in the conditional match).
    [Silent failures](../../gloss.html#silent-failure) like this are always hard to debug.

This version pulls the processing of each file out of the loop into a function of its own.
It also checks that `action` is one of the allowed flags before doing any processing, so that the program fails fast. We'll save it as `readings-05.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  action <- args[1]
  filenames <- args[-1]
  stopifnot(action %in% c("--min", "--mean", "--max"))
  
  for (f in filenames) {
    process(f, action)
  }
}

process <- function(filename, action) {
  dat <- read.csv(file = filename, header = FALSE)
  
  if (action == "--min") {
    values <- apply(dat, 1, min)
  } else if (action == "--mean") {
    values <- apply(dat, 1, mean)
  } else if (action == "--max") {
    values <- apply(dat, 1, max)
  }
  print(values)
}

main()
</code></pre></div>

This is four lines longer than its predecessor, but broken into more digestible chunks of 8 and 12 lines.

> **Tip:** R has a package named [argparse][argparse-r] that helps handle complex command-line flags (it utilizes a [Python module][argparse-py] of the same name).
We will not cover this package in this lesson but when you start writing programs with multiple parameters you'll want to read through the package's [vignette][].

[argparse-r]: http://cran.r-project.org/web/packages/argparse/index.html
[argparse-py]: http://docs.python.org/dev/library/argparse.html
[vignette]: http://cran.r-project.org/web/packages/argparse/vignettes/argparse.pdf

#### Challenges

  + Rewrite this program so that it uses `-n`, `-m`, and `-x` instead of `--min`, `--mean`, and `--max` respectively.
    Is the code easier to read?
    Is the program easier to understand?

  + Separately, modify the program so that if no parameters are given (i.e., no action is specified and no filenames are given), it prints a message explaining how it should be used.

  + Separately, modify the program so that if no action is given it displays the means of the data.

### Handling Standard Input

The next thing our program has to do is read data from standard input if no filenames are given so that we can put it in a pipeline, redirect input to it, and so on.
Let's experiment in another script, which we'll save as `count-stdin.R`:


<div class='out'><pre class='out'><code>count <- 0
lines <- readLines(con = file("stdin"))
for (line in lines) {
  count <- count + 1
}

print("lines in standard input:")
print(count)
</code></pre></div>

This little program reads lines from the program's standard input using `file("stdin")`.
This allows us to do almost anything with it that we could do to a regular file.
In this example, we passed it as an argument to the function `readLines`, which stores each line as an element in a vector.
Let's try running it from the Unix Shell as if it were a regular command-line program:


<pre class='in'><code>Rscript count-stdin.R < small-01.csv</code></pre>




<div class='out'><pre class='out'><code>[1] "lines in standard input:"
[1] 2
</code></pre></div>

A common mistake is to try to run something that reads from standard input like this:


<pre class='in'><code>Rscript count-stdin.R small-01.csv</code></pre>

i.e., to forget the `<` character that redirect the file to standard input.
In this case, there's nothing in standard input, so the program waits at the start of the loop for someone to type something on the keyboard.
We can type some input, but R keeps running because it doesn't know when the standard input has ended.
If you ran this, you can stop R by typing `ctrl`+`z`.

We now need to rewrite the program so that it loads data from `file("stdin")` if no filenames are provided.
Luckily, `read.csv` can handle either a filename or an open file as its first parameter, so we don't actually need to change `process`.
That leaves `main`, which we'll update and save as `readings-06.R`:


<div class='out'><pre class='out'><code>main <- function() {
  args <- commandArgs(trailingOnly = TRUE)
  action <- args[1]
  filenames <- args[-1]
  stopifnot(action %in% c("--min", "--mean", "--max"))
  
  if (length(filenames) == 0) {
    process(file("stdin"), action)
  } else {  
    for (f in filenames) {
      process(f, action)
    }
  }
}

process <- function(filename, action) {
  dat <- read.csv(file = filename, header = FALSE)
  
  if (action == "--min") {
    values <- apply(dat, 1, min)
  } else if (action == "--mean") {
    values <- apply(dat, 1, mean)
  } else if (action == "--max") {
    values <- apply(dat, 1, max)
  }
  print(values)
}

main()
</code></pre></div>

Let's try it out.
Instead of calculating the mean inflammation of every patient, we'll only calculate the mean for the first 10 patients (rows):


<pre class='in'><code>head inflammation-01.csv | Rscript readings-06.R --mean</code></pre>




<div class='out'><pre class='out'><code> [1] 5.450 5.425 6.100 5.900 5.550 6.225 5.975 6.650 6.625 6.525
</code></pre></div>

And now we're done: the program now does everything we set out to do.

#### Challenges

  + Write a program called `line-count.R` that works like the Unix `wc` command:
    *   If no filenames are given, it reports the number of lines in standard input.
    *   If one or more filenames are given, it reports the number of lines in each, followed by the total number of lines.

<div class="keypoints" markdown="1">
#### Key Points

*   Use `commandArgs(trailingOnly = TRUE)` to obtain a vector of the command-line arguments that a program was run with.
*   Avoid silent failures.
*   Use `file("stdin")` to connect to a program's standard input.
</div>
