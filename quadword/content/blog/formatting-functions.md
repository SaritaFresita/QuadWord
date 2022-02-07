---
title: Formatting functions
date: 2022-02-06T23:49:01+00:00
draft: false

image: /img/thumbs/fmtfuncs.jpg

description: Learn some C formatting functions

categories:
  - C
tags:
  - Programming
  - Beginners

type: post
---

Let's take a look at some C functions for formatting:

## sprintf

The `sprintf` (which stands for "string print formatted") function, is used to
write formatted output to a string. When using `sprintf`, you can combine
several data variables into a character array, is like `printf` but instead of
printing output on the console, you store the output data to a char buffer.

This is how its definition looks like:

```c
int sprintf (char *string, const char *format, ...);
```

The first parameter is a chat pointer that specifies where to send the output
(the buffer to put the data in), it terminates the string with a null character.

This is a little example on how you'd use it:

```c
sprintf (a_string, "health: %d", health);
```

We are printing to the `a_string` variable the value of "health: " and the value
of the `health` variable.

This function returns the number of characters stored in the string - _not
including the null terminator_. Sprintf will behave unpredictably if the stirng
to which it is printing overlaps any of its arguments, you need to make sure the
size of the buffer to be written to is large enough to avoid buffer overruns.

Sprintf is unsafe because it doesn't check the length of the destination buffer,
this can cause the function to overflow the destination buffer when the result
of the format string is unexpectedly long - this leads to security issues and
application instability -. Remmeber that overflows can cause unexpected results.

Let's look at an example:

```c
#include <stdio.h>

int
main ()
{
	char str [128] = { 0 };
	sprintf (str, "Hello, this is the number: %d", 55);
	puts (str);

	return 0;
}
```

And when you run that code, you'd get the following output:

```text
Hello, this is the number: 55
```

You can also do more things like:

```c
#include <stdio.h>

int
main ()
{
	char str [128] = { 0 };
	int a = 10;
	int b = 20;
	int c = a + b;

	sprintf (str, "The sum of %d and %d is %d", a, b, c);
	puts (str);

	return 0;
}
```

using this function.

There are some other functions such as `snprintf` that are like `strncpy` and
`strncmp`, where you can specify a limit for the limit to copy the characters.

## fprintf

The `fprintf` function is provided to perform the same operation as `printf`,
but on a specified file, it takes an additional argument, which is the FILE
pointer that identifies the file to which the data will be written to.

Its declaration looks like this:

```c
int fprintf (FILE *stream, const char *format, ...);
```

The following are a few examples of this function:

```c
fprintf (an_opened_file, "Hello, File!\n");
fprintf (stderr, "There's been an error on line: %d\n", line);
```

There's no much to say about this function, if you know the `printf` function,
you also know `fprintf`.

### fprintf and stderr

The reason why `stderr` exists is so that error messages can be logged to a
device or file other than where the normal output is written, it's particularly
desirable when the program's output is saved to a file, the normal output -
_stdout_ - is written into the file, but any system error messages will appear
in your window, you might want to write your own error messages to `stderr` for
this reason.

## fscanf

The `fscanf` function, is provided to perform the same operations as the `scanf`
function, but on a specified file. You can use this function to read formatted
input from a file and its arguments are the same as the `scanf` function, with
the different of a FILE pointer being required now. This is how its declaration
looks like:

```c
int fscanf (FILE *fp, const char *format, ...);
```

The additional arguments should point to already allocated objects of the type
specified by their corresponding format specifier within the format string. This
function returns the number of arguments that are successfully read and assigned
successfully and returns the value EOF if the end of the file is reached before
any of the conversion specifications have been processed.

For example:

```c
fscanf (a_file, "%d", &a_variable);
```

That call will read the next integer from the file `a_file` and stores it into
the `a_variable` variable.

## sscanf

The `sscanf` function allows you to read formatted data from a string, rather
than stdin or keyboard, this is how its declaration looks like:

```c
int sscanf (const char *str, const char *format, ...);
```

And this is how you'd use it:

```c
sscanf (buffer, "%s %d", name, &age);
```

The first argument is a pointer to the string where we want to read the data,
the rest of arguments of `sscanf` are the same as that of `scanf`.

## fscanf vs fgets and sscanf

If you use `fgets` and `sscanf` together, you must enter both values on the same
line. If you only use `fscanf` on stdin, it will read them off different lines
if it doesn't find the specified values on the first line you entered, if you
read a line that you are unable to parse with `sscanf` after having it read
using `fgets` your program will simply discard the line and move on, if you read
a line using `fscanf`, when it fails to convert fields, it leaves all the input
on the stream so, if you failed to read the input you wanted, you would have to
go and read all the data you want to ignore yourself.

You can just use `fscanf` by itself, however, you may be able to avoid some
problems by using `fgets` and `sscanf`.

## fflush

The `fflush` function is used to flush/clean a file or a buffer, it causes any
unwritten data in the output buffer to be sent to the output file, this process
is called "_flushing a buffer_", it cleans the buffer (making it empty) if it
has been loaded with any other data already. Its definition looks like this:

```c
int fflush (FILE *fp);
```

And this is an example of its usage:

```c
fflush (buffer);
```

In that above example, `buffer` is a temporary pointer variable which points to
the data, if the file is a null pointer, all output buffers are flushed.

**Note**: The effect of using `fflush` on an input stream is undefined.
