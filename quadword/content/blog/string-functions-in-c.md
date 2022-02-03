---
title: String Functions in C
date: 2022-02-03T16:47:29+00:00
draft: false

image: /img/strfunctions.png

description: Let's review some useful C functions to manipulate strings

categories:
  - C
tags:
  - Programming
  - Beginners

type: featured
---

Hello everyone, I am Lovelace, and in this entry we'll see some String I/O
Functions available in the C standard library.

## gets

The `gets` function stands for _get string_ and reads a line from the standard
input into a buffer, it reads until a terminating newline or EOF character is
found, takes one argument, a pointer to an array of chars where the string is
stored and returns `str` on success, and `NULL` on error or when end of file
occurs. This is how its declaration looks like:

```c
char *gets (char *str)
```

This function is now deprecated and should not be used anymore, it was removed
from C11, as a result, we won't talk much about it, there are two options for
`gets` that should be used instead: `fgets` and `getchar`, we will discuss
`fgets` from now on.

### fgets

The `fgets` function is used for reading entire lines of data from a
file/stream, it has a similar behaviour to `gets`, accepts two additional
arguments: the number of characters to read and an input stream. Its declaration
looks like this:

```c
char *fgets (char *buffer, int n, FILE *stream)
```

- **buffer** is a pointer to a character array where the line that is read in,
  will be stored.
- **n** is an integer value that represents the maximum number of characters to
  be stored into the buffer, including the NULL terminator.
- **stream** is the pointer to a file stream where characters are read from
  (`stdin` is a valid argument).

fgets reads characters from the specified file until a new line character has
been read or until n-1 character have been read (whichever occurs first). A NULL
character is written immediately after the last character read into the array,
it returns the value of `buffer` if the read is successful and returns the value
of `NULL` if an error occurs on the read or if an attempt is made to read past
the end of the file.

The fgets function won't copy the newline character, that's why it is possible
to read a partial line when using fgets and truncation of user input can be
detected because the input buffer will not contain a newline character. It also
protects against overflowing the string - _that's why it's not recommended for
performance reasons_ -.

Even though `fgets` is still part of the standard, it is also deprecated,
because the function cannot tell whether a null character is included in the
string it reads, if a null character is read, it will be stored in the string
along with the rest of the characters read. Since a null terminator terminates a
string, this will end your string prematurely, right before the first null
character.

Only use `fets` if you are sure that the data read cannot contain a null
character, otherwise, use `getline`.

Let's look at a snippet of C code where use the `fgets` function:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int
main ()
{
	char buf [255] = { 0 };
	int ch = 0;
	char *p = NULL;

	if (fgets (buf, sizeof (buf), stdin))
	{
		p = strchr (buf, '\n');
		if (p)
		{
			*p = '\0';
		}

		return 0; // Everything went successful
	}

	return 1; // There's been an error
}
```

In that code, we are basically reading 254 character (255 with the null
terminator) from the `stdin` file into the `buf` variable, if we are able to
read something from the `stdin` we will store the address of the `\n` character
in our `buf` (if any) in our `p` variable, if there's any newline character, we
will convert it into a null terminator.

## getline

The latest function for reading a string of text is `getline`, a new C library
function appearing around 2010 or so, this is the preferred method for reading
lines of text from a stream (including the stdio), the other functions like
`gets`, `fgets` and `scanf` are very unreliable, the `getline` function reads an
entire line from a stream, including the newline character. This function takes
three arguments and its declaration looks like this:

```c
ssize_t getline (char **buffer, size_t *size, FILE *stream);
```

- The first parameter is a double pointer to a block allocated with `malloc`
  or `calloc`, this function will automatically enlarge the block of memory as
  needed (using `realloc`), there will never be a shortage of space (that's why
  `getline` is so safe).
- The second parameter is a pointer to a variable of type `size_t`, this
  variable specifies the size in bytes of the block of memory pointed to by the
  first parameter, the address of the variable that holds the size of the input
  buffer, another pointer.
- The third parameter is simply the stream from which we will read the line
  from.

If an error occurs, such as the EOF being reached without reading any bytes,
`getline` returns -1, otherwise, it returns the number of characters read (up to
and including the newline, but not the final NULL character).

Let's look at an example:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int
main ()
{
	char *buffer = NULL;
	size_t bufsize = 32;
	size_t characters = 0;

	buffer = (char *)malloc (bufsize * sizeof (char));

	if (!buffer)
		exit (1);

	printf ("Type something: ");
	characters = getline (&buffer, &bufsize, stdin);

	printf ("%zu characters were read.\n", characters);
}
```

The string input is stored in `buffer`, we also have two other variables
`bufsize` and `characters`, which are required by getline. We allocate memory
for `buffer` (remember that if the input string is longer than `buffer`'s size,
getline will safely realloc the necessary amount of memory for it), now we just
call `getline` and we specify the buffer where to store the read string,
`bufsize` which contains the size of our buffer variable, the stream we want to
read from and finally, we store the total characters read in the `characters`
variable, finally, we print out the number of characters that were read.

## puts

The `puts` function is used to write a line to the screen, the most convenient
function for printing a simple message on the `stdout` - _it automatically
appends a new line_. It's simpler than `printf`, since you don't need to include
a newline character.

The different between puts and printf, is that when using printf the argument is
interpreted as a formatting string, the result will be often the same (except
for the added newline) if the string doesn't contain any control characters (%).

The puts function is safe and simple, but not very flexible as it doesn't give
us an option of formatting our string. This is how its declaration looks like:

```c
int puts (const char *string);
```

And this is a simple example using the `puts` function:

```c
#include <stdio.h>

int
main ()
{
	puts ("Hello, World!");
	return 0;
}
```

There's another function, similar to puts called `fputs`, this function accepts
a second parameter which is a custom file stream, therefore, you can write to a
file. This function DOES NOT add the newline, it only writes the characters in
the string and no more. This is what this function declaration looks like:

```c
int fputs (const char *buffer, FILE *fp);
```

Look at this simple example:

```c
#include <stdio.h>
#include <stdlib.h>

int
main ()
{
	FILE *fp = NULL;
	fp = fopen ("a_file.txt", "w");

	if (!fp)
		exit (1);

	fputs ("Hello, File!", fp);
	fclose (fp);

	return 0;
}
```
