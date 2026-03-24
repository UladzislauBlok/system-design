# Chapter 10.1 Batch Processing: Batch Processing with Unix Tools

Let’s start with a simple example. When a web server appends a line to a log file every time it serves a request, there is a lot of information in that single line. In order to interpret it—such as the default nginx access log format—you look at the log format definition. This allows you to extract details like the timestamp, the client IP address, the requested file URL, the response status, the byte size, and the client's web browser.

<br>

---

<br>

### Simple Log Analysis

While various tools can produce reports about website traffic, you can build your own using a chain of basic Unix tools. For example, to find the five most popular pages, you can pipe together commands to:

- Read the log file.
- Extract the specific URL field using `awk`.
- Alphabetically `sort` the list.
- Use `uniq -c` to filter out and count repeated adjacent lines.
- `sort` the counts in reverse numerical order.
- Use `head` to output just the top five lines.

```
cat /var/log/nginx/access.log |
  awk '{print $7}' |
  sort |
  uniq -c |
  sort -r -n |
  head -n 5

```

Although this looks obscure, it is incredibly powerful and can process gigabytes of log files in seconds. Instead of a chain of Unix commands, you could write a simple custom program to do the same thing. The primary difference between the two approaches becomes apparent with large files:

- In-memory aggregation: A custom script typically keeps an in-memory hash table. This works fine if the job's working set (the number of distinct URLs) fits entirely within the available memory.
- Sorting: The Unix pipeline relies on sorting the list. If the working set is larger than the available memory, the sorting approach makes efficient use of disks. The GNU Coreutils `sort` automatically handles larger-than-memory datasets by spilling to disk and parallelizing across CPU cores. It writes out chunks of sorted data to disk and merges them, benefiting from fast sequential I/O patterns.

<br>

---

<br>

### The Unix Philosophy

The ability to easily analyze data with a chain of commands is a key design idea of Unix. The idea of connecting programs with pipes—like a garden hose—became part of the Unix philosophy. Described in 1978, its core principles include:

    Make each program do one thing well.

    Expect the output of every program to become the input to another.

    Design and build software to be tried early.

    Use tools in preference to unskilled help.

To enable this composability, Unix relies on several specific architectural choices:

**A Uniform Interface**

If the output of one program is to become the input to another, they must use a compatible interface. In Unix, that uniform interface is a file (an ordered sequence of bytes). Because it is so simple, diverse things—actual files, device drivers, or TCP sockets—can share it and be plugged together. By convention, Unix programs treat this sequence of bytes as ASCII text, standardizing on the newline character (`\n`) as a record separator. This interoperability is rare today; even databases with the same data model often lack integration, leading to a Balkanization of data.

**Separation of Logic and Wiring**

Unix tools make heavy use of standard input (`stdin`) and standard output (`stdout`). The program itself doesn't worry about file paths; it just reads from `stdin` and writes to `stdout`. The shell user wires up the inputs and outputs using pipes. Separating the I/O wiring from the program logic makes it incredibly easy to compose small tools into bigger, customized pipelines.

**Transparency and Experimentation**

Unix tools are incredibly successful because they make data processing highly observable:

- Input files are treated as immutable, so you can run commands repeatedly without damaging the data.
- You can end a pipeline at any point and pipe the output into inspection tools to debug.
- You can write intermediate stages to a file and use that to restart later stages without rerunning the whole pipeline.

However, the biggest limitation of Unix tools is that they run only on a single machine—which is exactly where tools like Hadoop come in.
