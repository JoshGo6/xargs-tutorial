# How to use `xargs`

When one command acts on the output of the previous command and using a plain pipe won't work or is undesirable for some reason, you can chain the commands together using `xargs`. In the special case that the first command is `find`, however, you can use an alternative approach: the `find` command with the `-exec` option. This tutorial shows how to use `xargs`, including common options that control input splitting, batching, and parallel execution. Towards the end of the tutorial, we compare `xargs` and `find...-exec` in the special case where you have the option to use either approach.

This tutorial contains the following topics: 

- [Introduction and basic syntax](#Introduction%20and%20basic%20syntax)
- [Examples](#Examples)
  - [A basic example: archive files](#A%20basic%20example%20archive%20files)
  - [Ignore internal spaces in arguments](#Ignore%20internal%20spaces%20in%20arguments)
  - [Specify a delimiter](#Specify%20a%20delimiter)
  - [Process all arguments in a single line of output](#Process%20all%20arguments%20in%20a%20single%20line%20of%20output)
  - [Process all arguments in a specified number of lines of input](#Process%20all%20arguments%20in%20a%20specified%20number%20of%20lines%20of%20input)
  - [Specify the maximum number of inputs](#Specify%20the%20maximum%20number%20of%20inputs)
  - [Specify the maximum number of parallel processes](#Specify%20the%20maximum%20number%20of%20parallel%20processes)
- [Batching tradeoffs with `-I`](#Batching%20tradeoffs%20with%20`-I`)
- [Options](#Options)
- [`xargs` vs `find...-exec`](#`xargs`%20vs%20`find...-exec`)

## Introduction and basic syntax

Sometimes commands cannot or shouldn't operate on stdin, so simply piping output from one command to the next doesn't work. When you need to position the output of one command into a particular place in another command, you can use `xargs` in conjunction with a pipe. 

Invoke `xargs` by choosing one of the following two forms: 

```bash
<command1> | xargs <command2> # First form
<command1> | xargs -I {} <command2> {} # Second form
```

In the first form, `xargs` appends stdout from `<command1>` after `command2`. If you need the output to be put somewhere else, or if you need it to be used multiple times, use the second form, which contains the replacement token, `{}`. When `<command2>` is run, `xargs` replaces `{}` with stdout from `<command1>`.

## Examples

The following examples show commonly used options in `xargs`. These options can be used to control how the input stream is split into arguments, specify the number of arguments that are passed to a command from the input stream, and how many times to invoke a command in parallel.

### A basic example: archive files

A typical use of `xargs` involves taking the output of `find` and piping it to another command, as in the following example, where we create an archive file and then verify its contents:

```shellsession
$ ls
1.jpg  2.jpg  3.jpg  4.pdf
$ find . -type f | xargs tar -cf test.tar
$ ls
1.jpg  2.jpg  3.jpg  4.pdf  test.tar
$ tar -tvf test.tar 
-rw-rw-r-- josh/josh    976940 2025-12-23 16:04 ./2.jpg
-rw-rw-r-- josh/josh    182280 2025-12-23 16:04 ./1.jpg
-rw-rw-r-- josh/josh   1212841 2025-12-23 16:04 ./4.pdf
-rw-rw-r-- josh/josh    896583 2025-12-23 16:04 ./3.jpg
```

After the TAR file is created, we used `tar -tvf test.tar` to list the contents of the TAR file.

The rest of the examples show additional options that can be used with `xargs` to control splitting and batching of the input stream, as well as parallel processing.

### Ignore internal spaces in arguments

In the following example, the user creates a TAR file from files that have internal spaces in their filenames. There are at least two different ways to do this. This method uses `-print0` and `-0`.

```shellsession
$ ls
1.jpg   2.jpg   3.jpg   4.pdf  'one file.jpg'  'second file.jpg'
$ find . -type f -print0 | xargs -0 tar -cf test.tar
$ ls
1.jpg   2.jpg   3.jpg   4.pdf  'one file.jpg'  'second file.jpg'   test.tar
$ tar tvf test.tar 
-rw-rw-r-- josh/josh    976940 2025-12-23 17:19 ./second file.jpg
-rw-rw-r-- josh/josh    182280 2025-12-23 17:19 ./one file.jpg
-rw-rw-r-- josh/josh    976940 2025-12-23 16:04 ./2.jpg
-rw-rw-r-- josh/josh    182280 2025-12-23 16:04 ./1.jpg
-rw-rw-r-- josh/josh   1212841 2025-12-23 16:04 ./4.pdf
-rw-rw-r-- josh/josh    896583 2025-12-23 16:04 ./3.jpg
```

This is a typical way to handle arguments with spaces in the name, where you use `find` with `print0` to separate the arguments with a null character instead of a new line, and you use the `-0` option with `xargs` to indicate that input should be split on the null character, instead of on white space or new lines.

### Specify a delimiter

In the following example, we specify the delimiter `#` on which to split the input stream, where each argument except the final one is sandwiched between two `#` delimiters:

```shellsession
$ cat file.txt 
one#two#three#four#five
$ cat file.txt | xargs -d '#' echo
one two three four five

```

Note that `xargs` outputs a trailing blank line because the last argument was actually `five\n`.

### Process all arguments in a single line of output

By default, `xargs` splits on whitespace but will still feed all of the arguments at once to the command that comes after it, as in this example:

```shellsession
$ cat file.txt
one two three
four five six
seven eight nine
$ cat file.txt | xargs echo
one two three four five six seven eight nine
```

If you use the `-I {}` option, however, `echo` is now given one line of inputs at a time to process, as in the following example:

```shellsession
$ cat file.txt
one two three
four five six
seven eight nine
$ cat file.txt | xargs -I {} echo {}
one two three
four five six
seven eight nine
```

Now that we used `-I`, each invocation of `echo` is provided one line from `file.txt`, instead of the whole file.

### Process all arguments in a specified number of lines of input

We previously looked at using `xargs` to pass all arguments in a line of output at once. Instead of using `-I`, you can use `-L` to pass all arguments in a specified number of lines to a command, as in the following example, which feeds two lines of text at a time as a single argument to `echo`:

```shellsession
$ cat file.txt
one two three
four five six
seven eight nine
ten eleven twelve
$ cat file.txt | xargs -L2 echo
one two three four five six
seven eight nine ten eleven twelve
```

### Specify the maximum number of inputs

In this example, we use `-n5` to specify that a maximum of five arguments can be passed to each invocation of `echo` :

```shellsession
$ cat file.txt
one two three
four five six
seven eight nine
ten eleven twelve
$ cat file.txt | xargs -n5 echo
one two three four five
six seven eight nine ten
eleven twelve
```


### Specify the maximum number of parallel processes

So far, we've shown how to split the input stream into arguments, and how to specify which arguments are given to a command via `xargs`. In each of these cases, though, we've only invoked a command once, and when it finishes, `xargs` invokes it again with the next portion of the input stream. Now we'll invoke the command multiple times in parallel, using the `-P` option, as shown in the following example, where we run `gzip` up to 10 times in parallel, each time that `xargs` runs it:

```shellsession
find . -type f -print0 | xargs -0 -P 10 gunzip
```

## Batching tradeoffs with `-I`

Most often, you're likely to use `xargs` with `-I {}`, since this gives you the flexibility to use stdin at an arbitrary place of your choosing, when you build up your command. The tradeoff is that when you do this, each invocation of the command processes only one line of stdin, as in the following example, where we use the `-p` option for confirmation, so we can see exactly what `xargs` is about to execute:

```shellsession
$ find ~/Downloads/ -type f -mtime -1 | xargs -I {} -p mv {} ~/Downloads/old-files/
mv /home/josh/Downloads/01-28-2026.pdf /home/josh/Downloads/old-files/?...n
mv '/home/josh/Downloads/01-28-2026 (Copy 2).pdf' /home/josh/Downloads/old-files/?...n
mv '/home/josh/Downloads/01-28-2026 (Copy).pdf' /home/josh/Downloads/old-files/?...n
```

As you can see, only one `find` result is processed at a time, using this method. You can't use `-print0` with `-0` to force all arguments on to one line, since this option is incompatible with `-I`. Similarly, `-n` and `-L` are incompatible with `-I`, so your choices (if you want to use `xargs` instead of `find...-exec`) is to switch to a different form of `mv` which allows placement of the source at the end of the command so you can place all of the file names at the end of the command, or to have `mv` invoked as many times as there are files.
## Options

The following list summarizes the frequently used options with `xargs`:

- `-t`   Print each command prior to execution.
- `-p`   Ask for confirmation prior to execution.
- `-0`   Treat a null character as an end-of-argument character, instead of treating it as white space . When using this option with `xargs` on the output of `find`, you need to use `-print0` with the preceding `find` command. This option is mutually exclusive with `-I`. For an example, see [Ignore internal spaces in arguments](#Ignore%20internal%20spaces%20in%20arguments).
- `-d`   Specify the delimiter between entries. For example, since output from `ls` is delimited by `\n`, to properly process those files, use `xargs -d '\n' ...`. 

    This option implies that quotes are interpreted literally, instead of as special characters. Also, you can only use a single-byte character like a single letter or an escape sequence (such as `\n`) if you use this option. You can also use an octal or hex escape code for the delimiter, so the following are equivalent: 
    
    `find ... -print0 | xargs -0 ...`  
    `find ... -print0 | xargs -d '\0'...`     
    
     For an example, see [Specify a delimiter](#Specify%20a%20delimiter).  
     
- `-I {}` Replace the replacement string (specified here as `{}`, though it can be something else) with values from stdin. This allows placement of stdin wherever you choose, instead of at the end of the command. When this option is used, unquoted blanks aren't interpreted as the delimiter. Instead, the newline (**not the null character**) is the delimiter, which means that in the following example, `<cmd2>` is fed all arguments on one line of output from `<cmd1>` per invocation of `<cmd2>`, and during execution time, `{}` is replaced with stdout from `<cmd1>`:  
      
      `<cmd1> ... | xargs -I{} <cmd2> [options] {}`  
    
    This option is mutually exclusive with `-0`, `-L`, and `-n`. For an example, see [Process all arguments in a single line of output](#Process%20all%20arguments%20in%20a%20single%20line%20of%20output).  
        
- `-P <max-procs>`   Run up to `max-procs` commands in parallel. Use `-P 0` to run as many processes as possible simultaneously. For an example, see [Specify the maximum number of parallel processes](#Specify%20the%20maximum%20number%20of%20parallel%20processes).  
- `-L <n-lines>`   Pass at most `<n-lines>` of non-blank input lines to each command invocation. You can use this to split input on a particular number of lines of output from the previous command. This is mutually exclusive with `-n`. For an example, see [Process all arguments in a specified number of lines of input](#Process%20all%20arguments%20in%20a%20specified%20number%20of%20lines%20of%20input).  
- `-n <max-args>`   Pass at most `<max-args>` arguments to each command invocation. This controls batching by argument count, instead of by line count. This option is mutually exclusive with `-L`. For an example, see [Specify the maximum number of inputs](#Specify%20the%20maximum%20number%20of%20inputs).  

## `xargs` vs `find...-exec`

When the first command used is `find`, you have two options for executing the second command. You can either execute the second command in conjunction with `xargs`, or you can execute the second command using the `-exec` option that follows `find`. Each approach has advantages particular to specific use cases.

`xargs` has the following advantages over `find...-exec`:

- **Batching**: `xargs` can pass a large number of arguments to a single command invocation. This may run faster than when using `find` for commands that are CPU-intensive when starting up.
- **Resource limits**: `xargs` divides the list of arguments into groups small enough to be acceptable to the executed command, respecting the system's command-line length limit.

`find...-exec` has the following advantages over `xargs`:

- **Readability**: There's no pipe, making the two commands easier to read for simple cases.
- **Single process**: The whole operation is a single process, which can be easier to manage and terminate.
- **Safety**: Even when filenames contain spaces, newlines, or other special characters, `-exec` doesn't need additional options to handle these files properly. You just use it as you would in any other case.


