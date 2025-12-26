# xargs, exec, and piping

This article contains the following topics:


When one command acts on the output of the previous command and using a plain pipe won't work or is undesirable for some reason, you can chain the commands together using `xargs`. If the first command is `find`, you can use `find...-exec`. This tutorial dives into both approaches and also compares them.

This tutorial contains the following topics: 



## xargs

Sometimes commands cannot or shouldn't operate on stdin, so simply piping output from one command to the next doesn't work. When you need to position the output of one command into a particular place in another command, you can use `xargs` in conjunction with a pipe. 

Invoke `xargs` by choosing one of the following two syntaxes: 

```bash
<command1> | xargs <command2> # First form
<command1> | xargs -I {} <command2> {} # Second form
```

In the first form, `xargs` appends the piped input after `command2`. If you need the output to be put somewhere else, or if you need it to be used multiple times, use the second form, which contains the replacement token, `{}`. When `<command2>` is run, `xargs` replaces `{}` with the output from `<command1>`.

> A very common use case is `find ... | xargs`. By default, `find` separates its outputs with new lines (`\n`). Also by default, `xargs` interprets unquoted spaces and newline characters as delimiters. If your output from `find` contains spaces, you have several options:
> 
>  - Use the `-print0` option with `find`, and the `-0` option with `xargs` to ensure that `xargs` doesn't break an output from `find` on spaces or newline characters.
>  - Use the `print0` option with `find`, and the `-d '\0'` option with `xargs`. 
> 
> The examples in [Examples](#Examples) illustrate this and other options.

### Constructing arguments through splitting and delimiters

`xargs` performs both splitting and batching of arguments. *Splitting* refers to how `xargs` divides stdin into individual arguments.  `xargs` splits arguments based on new lines and spaces. If you wish to retain this default behavior, as with other commands, you can use quotes around arguments that contain internal spaces.

So far, we've spoken about the default splitting behavior that's used to divide the input stream into arguments. As an example of this default, `"file1 file2 file3"` gets split into 3 arguments by default. You can change this by splitting, though, by specifying a delimiter other than whitespace.

When you change the delimiter from the default (whitespace) to something else like newline (but not a simple space) or null or a different character, `xargs` doesn't interpret quotes as special characters. Instead, they're treated as literals, and spaces are considered part of the argument (since something else is now the delimiter).

In the examples, we'll show how to change the default splitting behavior.

### Controlling the number of arguments via batching

While splitting determines exactly what each argument is (and therefore the total number of arguments, by specifying the delimiter), *batching* refers to how many of the arguments `xargs` passes to each command invocation. Depending on the options you choose, after splitting `file1 file2 file3` into 3 arguments, `xargs` could run any of the following:

- `command file1 file2 file3` (all 3 in one batch)
- `command file1`, then `command file2`, then `command file3` (one file per batch)
- `command file1 file2`, then `command file3`

The default batching behavior is to provide as many arguments as possible to the second command. In the examples, we'll show how to change the default batching behavior.

### Invoking the `xargs` command in parallel

**Invocations** refers to how many commands `xargs` will run in parallel, which is determined by the optional `-P` parameter. By default, xargs Invokes its command one time before invoking it again.

### Examples

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

The rest of the examples show additional options that can be used with `xargs` to control splitting, batching, and invocation. Many of the examples are constructed from the file `raven.txt`, which contains the first few lines from the poem "The Raven."

```consolte-session
$ cat raven.txt 
Once upon a midnight dreary, while I pondered, weak and weary,
Over many a quaint and curious volume of forgotten lore,
While I nodded, nearly napping, suddenly there came a tapping,
As of some one gently rapping, rapping at my chamber door. “
```

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

### Options

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

### xargs with shell builtins

In order to use `xargs` with a shell builtin, you need to invoke `bash -c` or `sh -c`:

```bash
find . -type f -name "v3*" -exec bash -c 'cd "{}" && pwd' \;
```

## exec

See [[find with exec and ok]].

## `xargs` vs `-exec`

Both `xargs` and `find ... -exec` can be used to execute commands on items found by `find`, but they work in different ways and are better suited for different scenarios. Here are some considerations to help you decide when to use each:

### xargs

1. **Batch Execution**: `xargs` can pass a large number of arguments to a single command invocation, which can make it faster for commands that are expensive to start up.
    
2. **Flexibility**: `xargs` is a separate utility that can be used with any command that reads from standard input. It's not tied to `find`, so you can use it in a wider variety of contexts.
    
3. **Delimiter Support**: With `-0` or `-d`, `xargs` can handle filenames or arguments that contain spaces, newlines, or other special characters.
    
4. **Error Handling**: `xargs` has options for more complex error handling (`-P` for parallelism, `--arg-file`, etc.).
    
5. **Resource Limits**: `xargs` takes care of dividing the list into sublists small enough to be acceptable to the executed command, respecting the system's command-line length limit.
    
6. **Custom Formatting**: You can use `xargs` to insert arguments at specific positions in the command line using the `-I` flag.
    

### find ... -exec

1. **Simplicity**: `-exec` is simpler to use for straightforward tasks and avoids a pipeline, making it easier to read and understand for simple use-cases.
    
2. **Atomicity**: When using `find ... -exec`, the whole operation is a single process, which can be easier to manage and terminate.
    
3. **Safety**: The `-exec` command works more predictably with filenames that contain spaces, newlines, or other special characters, without the need for additional options like `-0` with `xargs`.
    
4. **Per-File Operations**: If you're performing an operation that must be executed separately for each file, `find ... -exec` can often be clearer and easier.
    
5. **Built-in to `find`**: Since it's a built-in action for `find`, you don't need to worry about compatibility between two separate commands.
    
6. **Command Chaining**: `find`'s `-exec` can be combined with other `find` options and tests more naturally.
    

### Summary

- Use `xargs` when you need more flexibility, when you're dealing with a large number of files, or when you need more advanced features like parallelism or custom delimiters.
    
- Use `find ... -exec` when you're doing something simple and want a self-contained command, or when you're performing an operation that must be executed separately for each file and you want to keep things simple and readable.
    

Both tools are powerful, and your specific needs will dictate which is the better choice for a given situation.


