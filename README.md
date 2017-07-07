self extracting tar archives
============================


usage
-----

```
./selfextracttar file1 file2 file3...
```

it will generate an executable file called `archive.sh`, which can be extracted by either

- running it (e.g. `./archive.sh`), or
- with tar (e.g. `tar xf archive.sh`)



how it works
------------

the tar format is just a list of {header+data} blocks.
the header is 512 bytes long, the data is always padded to the next 512 bytes boundary.

to embed an extracting script in a tar we abuse two facts:

- the first field in the header is the filename of the current file, and
- posix shells parse files line by line

we start by putting our shebang in the filename field of the first file.  then we add __a newline and a `#`__, to make the following data (until the next newline) a comment according to a posix shell.

we must also make sure that no `\0` appears anywhere in the script, as some shells will choke on those and stop parsing.  this means that our shell script must call `exit` to stop processing the rest of the archive.

the tar format is quite laxly defined, and most implementation will try to be permissive.  the standard requires some fields to be `\0`-terminated, but apparently no real world tar enforces this.  

the script we embed is trivial, just skip the first file in the archive with `dd` and pipe the rest to `tar x`

---

now we have a bash script that can extract itself, and that technically is a valid tar archive.  `tar t` will show that the first file is called `#!/bin/sh`, which means a file called `sh` in the directory `#!/bin`.  `tar f` would happily extract such file and create the needed directories.

but hey, a modern tar will probably handle a lot more file types than the old primitive implementations in ancient unix.  maybe there's a way to mark a file as deleted?

turns out, there isn't.  gnu tar implements deletion by rewriting the archive without that file.  but... we don't actually need to mark it as deleted, just a way to avoid extracting it.

this is way easier to do, as most modern implementations share a few common extensions.  the following code is from gnu tar's `tar.h`:

```
/* This file is a tape/volume header.  Ignore it on extraction.  */
#define GNUTYPE_VOLHDR 'V'
```

by setting the type field to `'V'` in the header, we create a tar file with a file that will be skipped when extracting.  sadly, `tar t` will still list it as the first file in the archive.

another type that will cause the current file to be skipped is `'K'`, which means that the current file's content are the link name of the next file, which is too long to fit in the next file's link name field.  this will get skipped when listing with `tar t`.

the first file will be type `'K'`, then the next file will be type `'V'`.  by combining the two, we achieve the holy grail!... sort of

`tar t` will still try to list the first file, so we give it an empty file name.  this causes `tar t` to print a leading empty line,  and`tar tv` to list the first file as a volume header, but this is about the best one can do.   for most purposes, this file will only appear to contain the actual content we archived.
