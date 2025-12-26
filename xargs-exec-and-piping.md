# xargs, exec, and piping

This article contains the following topics:


When one command acts on the output of the previous command and using a plain pipe won't work or is undesirable for some reason, you can chain the commands together using `xargs`. If the first command is `find`, you can use `find...-exec`. This tutorial dives into both approaches and also compares them.

This tutorial contains the following topics: 

## Introduction and basic syntax

Sometimes commands cannot or shouldn't operate on stdin, so simply piping output from one command to the next doesn't work. When you need to position the output of one command into a particular place in another command, you can use `xargs` in conjunction with a pipe. 

Invoke `xargs` by choosing one of the following two syntaxes: 

```bash
<command1> | xargs <command2> # First form
<command1> | xargs -I {} <command2> {} # Second form
```

In the first form, `xargs` appends the piped input after `command2`. If you need the output to be put somewhere else, or if you need it to be used multiple times, use the second form, which contains the replacement token, `{}`. When `<command2>` is run, `xargs` replaces `{}` with the output from `<command1>`.

## Examples

A typical use of `xargs` involves taking the output of `find` and piping it to another command, as in the following example, where we create an archive file and then verify its contents:

```shell-session
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

**Create a TAR from files with internal spaces**

In the following example, the user creates a TAR file from files that have internal spaces in their filenames. There are at least two different ways to do this. This method uses `-print0` and `0`.

```shell-console
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

**Specify a delimiter**

In the following example, we specify a delimiter, `#` to split on:

```shell-console
j$ cat file.txt 
one#two#three#four#five
$ cat file.txt | xargs -d '#' echo
one two three four five

$ 
```

Note that `xargs` output a trailing blank line because the last argument was actually `five\n`.

**Process all arguments in a single line of output**

By default, `xargs` splits on whitespace but will still feed all of the arguments to the command that comes after it, as in this example:

```shell-console
$ cat file.txt
one two three
four five six
seven eight nine
$ cat file.txt | xargs echo
one two three four five six seven eight nine
```

If you use the `-I {}` option, however, `echo` is now given one line of inputs at a time to process, as in the following example:

```shell-console
$ cat file.txt
one two three
four five six
seven eight nine
$ cat file.txt | xargs -I {} echo {}
one two three
four five six
seven eight nine
```

Just as with the original `cat` command, now that we used `-I`, each line of the file becomes one argument to `echo`, instead of the file being split on regular whitespace characters.

**Process all arguments in a specified number of lines of input**

We previously looked at using `xargs` to pass all arguments in a line of output at once. Instead of using `-I`, you can use `-L` to pass all arguments in a specified number of lines to a command, as in the following example, which feeds two lines of text at a time as a single argument to `echo`:

```shell-console
$ cat file.txt
one two three
four five six
seven eight nine
ten eleven twelve
$ cat file.txt | xargs -L2 echo
one two three four five six
seven eight nine ten eleven twelve
```

**Specify the maximum number of inputs**

In this example, we use `-n5` to specify that a maximum of five arguments can be passed to `echo` at once:

```shell-console
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

## Options

The following list summarizes the frequently used options with `xargs`:

- `-t`   Print each command prior to execution
- `-p`   Ask for confirmation prior to execution
- `-0`   Treat null character as the end-of-argument character, instead of whitespace. When using this option with `xargs` on the output of `find`, you need to use `-print0` with the preceding `find` command. See [[Finding files/Find & xargs with spaces in file names]]. This is mutually exclusive with `-I`.
- `-d`   Specify the delimiter between entries. For example, since output from `ls` is delimited by `\n`, to properly process those files, use `xargs -d '\n' ...`. 

    This option implies that quotes are interpreted literally, instead of as special characters. Also, you can only use a single-byte character like a single letter or an escape sequence (such as `\n`) if you use this option. You can also use an octal or hex escape code for the delimiter, so the following are equivalent: 
    
    `find ... -print0 | xargs -0 ...`  
    `find ... -print0 | xargs -d '\0'...`     
     
- `-I {}` Replace the replacement string (specified here as `{}`, though it can be something else) with values from stdin. This allows placement of stdin at a place of your choosing, instead of as the last portion of the built-up command. When this option is used, unquoted blanks aren't interpreted as the dellimiter. Instead, the newline (**not the null charactere**) is the delimiter, which means that in the following example, `cmd2` is fed all arguments on one line of output from `cmd1` per invocation of `cmd2`, and during execution time, `{}` is replaced with output from `cmd1`:  
      
      `<cmd1> ... | xargs -I{} <cmd2> [options] {}`  
    
    This is mutually exclusive with `-0`.
- `-P <max-procs>`   Run up to `max-procs` commands in parallel. Use `-P 0` to run as many processes as possible simultaneously. Example: `find . -name "*.txt" | xargs -P 4 gzip` compresses 4 files at once.
- `-L <n-lines>`   Pass at most `<n-lines>` of non-blank input lines to each command invocation. You can use this to limit the number of lines of input that an invocation of a command processes. For instance, `cat urls.txt | xargs -L 1 curl` only allows `curl` to process one line from at a time from `urls.txt`. This is mutually exclusive with `-n`.
- `-n <max-args>`   Pass at most `<max-args>` arguments to each command invocation. This controls batching by argument count, instead of by line count. Mutually exclusive with `-L` (last one specified takes effect). Example: `echo "1 2 3 4 5 6" | xargs -n 2 echo` outputs three pairs of numbers.

## `xargs` vs `find...-exec`

When chaining commands `<cmd1>` and `<cmd2>, if <cmd2> acts on stdin, you can use a simple pipe, ` as in `<cmd1> | <cmd2>`. When `<cmd2>` takes a positional argument, instead of acting on stdin, you can use `xargs`. An example is `mv`, which requires you to first specify a source, and then a destination. 

Both `xargs` and `find ... -exec` can be used to execute commands on the output from `find`, but they work in different ways, and each one is better suited to particular scenarios.

The following are some of the advantages that `xargs` over `find...-exec`:

- **Batch execution**: It can pass a large number of arguments to a single command invocation, which can be more efficient for commands that are expensive to start up.
- **Resource Limits**: It divides the list of arguments into sublists small enough to be acceptable to the executed command, respecting the system's command-line length limit.

 find ... -exec:

1. **Simplicity**: `-exec` is simpler to use for straightforward tasks and avoids a pipeline, making it easier to read and understand for simple use-cases.
    
2. **Atomicity**: When using `find ... -exec`, the whole operation is a single process, which can be easier to manage and terminate.
    
3. **Safety**: The `-exec` command works more predictably with filenames that contain spaces, newlines, or other special characters, without the need for additional options like `-0` with `xargs`.
    
4. **Per-File Operations**: If you're performing an operation that must be executed separately for each file, `find ... -exec` can often be clearer and easier.
    
5. **Built-in to `find`**: Since it's a built-in action for `find`, you don't need to worry about compatibility between two separate commands.
    
6. **Command Chaining**: `find`'s `-exec` can be combined with other `find` options and tests more naturally.


