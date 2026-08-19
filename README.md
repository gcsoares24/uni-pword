# pword: a multi-process CLI tool for counting, locating and matching a word across text files

![Python](https://img.shields.io/badge/Python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Multiprocessing](https://img.shields.io/badge/multiprocessing-blue?style=for-the-badge)

> 📖 Quick note in Portuguese: You can also read this README in Portuguese. To do so, just access [here](README.pt.md).

## About the project

`pword` is a command-line tool that spawns multiple **child processes** to search for a word across one or more text files concurrently. Depending on how many files and processes are involved, it either distributes whole files among the child processes or splits a single large file into chunks, so the work is done in parallel instead of sequentially. It periodically logs progress (partial counts, processed/pending files) at a configurable interval, and gracefully handles `SIGINT` (Ctrl+C) to stop processing in a controlled way.

It was developed for the **Operating Systems** (Sistemas Operativos) course, group SO-TI-13, by Guilherme Soares, Duarte Soares and Vitória Correia.

### Features
- Concurrent word search/count across multiple files using `multiprocessing.Process`
- Three modes of operation, selected with `-m`:
  - `c` (count): counts every occurrence of the word as a substring of any token in the text, using a single shared counter protected by a `Lock`
  - `l` (locate): counts and collects the unique lines that contain the word as a substring, one counter per child process plus a `Queue` used to gather the matched lines
  - `i`: counts exact occurrences of the word as a whole token (i.e. `word == token`, not just a substring match), one counter per child process aggregated through a `Queue`
- Automatic work distribution across child processes:
  - if there are more processes than files, the number of processes is reduced to match the number of files
  - if there are more files than processes, files are distributed alternately (round-robin) among the processes
  - if a single file is given with multiple processes, the file is split into line-based chunks, one per process
- Configurable progress logging interval (`-i`), printed either to `stdout` or written to a log file (`-d`), including partial counts and processed/remaining file counts
- Graceful `SIGINT` handling: child processes ignore `SIGINT` directly (only the parent process reacts to it); the parent sets a shared flag that stops further file processing, and the program exits early via `sys.exit(0)` if the signal arrives before files were divided or before the child processes were created

### Usage

```
./pword -m <c|l|i> -p <n_processes> -i <interval> -d <log_file|stdout> -w <word> file1 [file2 ...]
```

Example commands:
1. `./pword -m c -p 2 -d testLog.log -w palavra testFiles/file.txt`
2. `./pword -m l -w ola -p 1 testFiles/file.txt`
3. `./pword -m i -p 12 -w palavra testFiles/file.txt testFiles/file.txt`
4. `./pword -p 2 -i 1 -w palavra testFiles/file.txt`
5. `./pword -m c -i 1 -d testLog.log -w palavra testFiles/file1.txt testFiles/file2.txt`

Flags:
- `-m`: mode of operation, `c`, `l` or `i` (defaults to `c`)
- `-p`: number of child processes to spawn (defaults to `1`)
- `-w`: word to search for (required)
- `-i`: log registration interval in seconds
- `-d`: destination file for the progress log (defaults to `stdout` if not provided)

**Log registration interval**
- If `-i` is not provided, the default value will be 3 seconds.
- If a value is provided, it is used instead. For example, `-i 1` sets the interval to 1 second.
- While any child process is still running, the parent periodically aggregates the counters and prints (or writes to the log file) the partial word count, the number of files already processed and the number of files still pending.

**Approach for dividing files**
- If there are more child processes than files, the number of child processes will be equal to the number of files.
- If there are more files than child processes, the files will be distributed among the child processes. For example: if there are 5 files and 2 child processes, the files will be assigned alternately starting with the first process, until all files are distributed. In this case, the first child process will handle 3 files and the second one 2 files.
- If there is only one file and multiple child processes, the file is split into chunks (by line ranges) corresponding to the number of child processes.

**SIGINT (Ctrl+C) behavior**
- Child processes explicitly ignore `SIGINT`, so only the parent process reacts to it.
- If `SIGINT` is triggered before the division of files (or chunks) into work items, `sys.exit(0)` is triggered and a message is displayed, since the process is interrupted before file division.
- If `SIGINT` is triggered after division but before the child processes are created/distributed the work, `sys.exit(0)` is triggered and a message is displayed, since the process is interrupted before file distribution.
- If `SIGINT` is triggered while child processes are already running, the shared flag is set to stop them from processing any further files/chunks, letting the program wind down in a controlled way.

### Tech stack
- Python 3
- `multiprocessing` (`Process`, `Value`, `Lock`, `Array`, `Queue`) for parallel processing and inter-process communication
- `signal` module for `SIGINT` handling
- Bash (the `pword` launcher script handles argument parsing/validation with `getopts` before invoking `pword.py`)
