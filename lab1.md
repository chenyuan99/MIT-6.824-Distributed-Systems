
<!DOCTYPE html>
<html>
<head>
<link rel="stylesheet" href="../style.css" type="text/css">
</head>
<body>
<div align="center">
<h2><a href="../index.html">6.5840</a> - Spring 2024</h2>
<h1>6.5840 Lab 1: MapReduce</h1>

<CENTER>
<p>
  <b><a href="collab.html">Collaboration policy</a></b> //
  <b><a href="submit.html">Submit lab</a></b> //
  <b><a href="go.html">Setup Go</a></b> //
  <b><a href="guidance.html">Guidance</a></b> //
  <b><a href="https://piazza.com/mit/spring2024/65840">Piazza</a></b>
</p>
</CENTER>

</div>
<hr>

<h3>Introduction</h3>

<p>
In this lab you'll build a MapReduce system.
You'll implement a worker process that calls application Map and Reduce
functions and handles reading and writing files,
and a coordinator process that hands out tasks to
workers and copes with failed workers.
You'll be building something similar to the
<a href="http://research.google.com/archive/mapreduce-osdi04.pdf">MapReduce paper</a>.
(Note: this lab uses "coordinator" instead of the paper's "master".)

<h3>Getting started</h3>

<p>You need to <a href="go.html">setup Go</a> to do the labs.

<p>
Fetch the initial lab software with
<a href="https://git-scm.com/">git</a> (a version control system).
To learn more about git, look at the
<a href="https://git-scm.com/book/en/v2">Pro Git book</a> or the
<a href="http://www.kernel.org/pub/software/scm/git/docs/user-manual.html">git user's manual</a>.

<pre>
$ git clone git://g.csail.mit.edu/6.5840-golabs-2024 6.5840
$ cd 6.5840
$ ls
Makefile src
$
</pre>

<p>
We supply you with a simple sequential mapreduce implementation in
<tt>src/main/mrsequential.go</tt>. It runs the maps and reduces one at a
time, in a single process. We also
provide you with a couple of MapReduce applications: word-count
in <tt>mrapps/wc.go</tt>, and a text indexer
in <tt>mrapps/indexer.go</tt>. You can run
word count sequentially as follows:

<pre>
$ cd ~/6.5840
$ cd src/main
$ go build -buildmode=plugin ../mrapps/wc.go
$ rm mr-out*
$ go run mrsequential.go wc.so pg*.txt
$ more mr-out-0
A 509
ABOUT 2
ACT 8
...
</pre>

<p>
<tt>mrsequential.go</tt> leaves its output in the file <tt>mr-out-0</tt>.
The input is from the text files named <tt>pg-xxx.txt</tt>.

<p>
Feel free to borrow code from <tt>mrsequential.go</tt>.
You should also have a look at <tt>mrapps/wc.go</tt> to see what
MapReduce application code looks like.

<p>
For this lab and all the others, we might issue updates to the code we
provide you. To ensure that you can fetch those updates and easily
merge them using <tt>git pull</tt>, it's best to leave the code we
provide in the original files. You can add to the code we provide as
directed in the lab write-ups; just don't move it. It's OK to put your
own new functions in new files.

<h3>Your Job <script>g("moderate/hard")</script></h3>

Your job is to implement a distributed MapReduce, consisting of
two programs, the coordinator and the worker. There will be
just one coordinator process, and one or more worker processes executing in
parallel. In a real system the workers would run on a bunch of
different machines, but for this lab you'll run them all on a single machine.
The workers will talk to the coordinator via RPC. Each worker process will,
in a loop, ask
the coordinator for a task, read the task's input from one or more files,
execute the task, write the task's output to one
or more files, and again ask the coordinator for a
new task. The coordinator should notice if a worker hasn't completed
its task in a reasonable amount of time (for this lab, use ten
seconds), and give the same task to a different worker.

<p>
We have given you a little code to start you off. The "main" routines for
the coordinator and worker are in <tt>main/mrcoordinator.go</tt> and <tt>main/mrworker.go</tt>;
don't change these files. You should put your implementation in <tt>mr/coordinator.go</tt>,
<tt>mr/worker.go</tt>, and <tt>mr/rpc.go</tt>.

<p>
Here's how to run your code on the word-count MapReduce
application. First, make sure the word-count plugin is
freshly built:
<pre>
$ go build -buildmode=plugin ../mrapps/wc.go
</pre>

In the <tt>main</tt> directory, run the coordinator.
<pre>
$ rm mr-out*
$ go run mrcoordinator.go pg-*.txt
</pre>

The <tt>pg-*.txt</tt> arguments to <tt>mrcoordinator.go</tt> are
the input files; each file corresponds to one "split", and is the
input to one Map task.

<p>
In one or more other windows, run some workers:
<pre>
$ go run mrworker.go wc.so
</pre>

When the workers and coordinator have finished, look at the output
in <tt>mr-out-*</tt>. When you've completed the lab, the
sorted union of the output files should match the sequential
output, like this:
<pre>
$ cat mr-out-* | sort | more
A 509
ABOUT 2
ACT 8
...
</pre>

<p>
We supply you with a test script in <tt>main/test-mr.sh</tt>. The tests
check that the <tt>wc</tt> and <tt>indexer</tt> MapReduce applications
produce the correct output when given the <tt>pg-xxx.txt</tt> files as
input. The tests also check that your implementation runs the Map and
Reduce tasks in parallel, and that your implementation recovers from
workers that crash while running tasks.

<p>
If you run the test script now, it will hang because the coordinator never finishes:
</p>

<pre>
$ cd ~/6.5840/src/main
$ bash test-mr.sh
*** Starting wc test.
</pre>

<p>
You can change <tt>ret := false</tt> to true in the Done function in <tt>mr/coordinator.go</tt>
so that the coordinator exits immediately. Then:
</p>

<pre>
$ bash test-mr.sh
*** Starting wc test.
sort: No such file or directory
cmp: EOF on mr-wc-all
--- wc output is not the same as mr-correct-wc.txt
--- wc test: FAIL
$
</pre>

<p>
The test script expects to see output in files named <tt>mr-out-X</tt>, one
for each reduce task. The empty implementations of <tt>mr/coordinator.go</tt>
and <tt>mr/worker.go</tt> don't produce those files (or do much of
anything else), so the test fails.

<p>
When you've finished, the test script output should look like this:

<pre>
$ bash test-mr.sh
*** Starting wc test.
--- wc test: PASS
*** Starting indexer test.
--- indexer test: PASS
*** Starting map parallelism test.
--- map parallelism test: PASS
*** Starting reduce parallelism test.
--- reduce parallelism test: PASS
*** Starting job count test.
--- job count test: PASS
*** Starting early exit test.
--- early exit test: PASS
*** Starting crash test.
--- crash test: PASS
*** PASSED ALL TESTS
$
</pre>

<p>
You may see some errors from the Go RPC package that look like
<pre>
2019/12/16 13:27:09 rpc.Register: method "Done" has 1 input parameters; needs exactly three
</pre>
Ignore these messages; registering the coordinator as an <a
    href=https://golang.org/src/net/rpc/server.go>RPC server</a> checks if all its 
methods are suitable for RPCs (have 3 inputs); we know that <tt>Done</tt> is not called via RPC.

<p>
Additionally, depending on your strategy for terminating worker processes, you may see some errors of the form
<pre>
2024/02/11 16:21:32 dialing:dial unix /var/tmp/5840-mr-501: connect: connection refused
</pre>
It is fine to see a handful of these messages per test; they arise when the worker is unable to contact the coordinator RPC server after
the coordinator has exited.

<p>
<h3>A few rules:</h3>

<ul>

<li>
The map phase should divide the intermediate keys into buckets for
<tt>nReduce</tt> reduce tasks,
where <tt>nReduce</tt> is the number of reduce tasks -- the argument that
<tt>main/mrcoordinator.go</tt> passes to <tt>MakeCoordinator()</tt>.
Each mapper should create <tt>nReduce</tt> intermediate files for
consumption by the reduce tasks.

<li> The worker implementation should put the output of the X'th
reduce task in the file <tt>mr-out-X</tt>.

<li> A <tt>mr-out-X</tt> file should contain one line per Reduce
function output. The line should be generated with the Go <tt>"%v %v"</tt>
format, called with the key and value. Have a look in <tt>main/mrsequential.go</tt>
for the line commented "this is the correct format".
The test script will fail if your implementation deviates too much from this format.

<li> You can modify <tt>mr/worker.go</tt>, <tt>mr/coordinator.go</tt>, and <tt>mr/rpc.go</tt>.
You can temporarily modify other files for testing, but make sure your code works
with the original versions; we'll test with the original versions.

<li> The worker should put intermediate Map output in files in the current
directory, where your worker can later read them as input to Reduce tasks.

<li> <tt>main/mrcoordinator.go</tt> expects <tt>mr/coordinator.go</tt> to implement a
<tt>Done()</tt> method that returns true when the MapReduce job is completely finished;
at that point, <tt>mrcoordinator.go</tt> will exit.

<li> When the job is completely finished, the worker processes should exit.
A simple way to implement this is to use the return value from <tt>call()</tt>:
if the worker fails to contact the coordinator, it can assume that the coordinator has exited
because the job is done, so the worker can terminate too. Depending on your
design, you might also find it helpful to have a "please exit" pseudo-task
that the coordinator can give to workers.

</ul>

<h3>Hints</h3>

<ul>

<li>The <a href="guidance.html">Guidance page</a> has some 
  tips on developing and debugging.

<li> One way to get started is to modify <tt>mr/worker.go</tt>'s
<tt>Worker()</tt> to send an RPC to the coordinator asking for a task. Then
modify the coordinator to respond with the file name of an as-yet-unstarted
map task. Then modify the worker to read that file and call the
application Map function, as in <tt>mrsequential.go</tt>.

<li> The application Map and Reduce functions are loaded at run-time
using the Go plugin package, from files whose names end in <tt>.so</tt>.

<li> If you change anything in the <tt>mr/</tt> directory, you will
probably have to re-build any MapReduce plugins you use, with
something like <tt>go build -buildmode=plugin ../mrapps/wc.go</tt>

<li> This lab relies on the workers sharing a file system.
That's straightforward when all workers run on the same machine, but would require a global
filesystem like GFS if the workers ran on different machines.

<li> A reasonable naming convention for intermediate files is <tt>mr-X-Y</tt>,
where X is the Map task number, and Y is the reduce task number.

<li> The worker's map task code will need a way to store intermediate
key/value pairs in files in a way that can be correctly read back
during reduce tasks. One possibility is to use Go's <tt>encoding/json</tt> package. To
write key/value pairs in JSON format to an open file:
<pre>
  enc := json.NewEncoder(file)
  for _, kv := ... {
    err := enc.Encode(&kv)
</pre>
and to read such a file back:
<pre>
  dec := json.NewDecoder(file)
  for {
    var kv KeyValue
    if err := dec.Decode(&kv); err != nil {
      break
    }
    kva = append(kva, kv)
  }
</pre>

<li> The map part of your worker can use the <tt>ihash(key)</tt> function
(in <tt>worker.go</tt>) to pick the reduce task for a given key.

<li> You can steal some code from <tt>mrsequential.go</tt> for reading
Map input files, for sorting intermedate key/value pairs between the
Map and Reduce, and for storing Reduce output in files.

<li> The coordinator, as an RPC server, will be concurrent; don't forget
to lock shared data.

<li> Use Go's race detector, with <tt>go run -race</tt>.
<tt>test-mr.sh</tt> has a comment at the start that tells you
how to run it with <tt>-race</tt>.
When we grade your labs, we will <b>not</b> use the
race detector. Nevertheless, if your code has races, there's
a good chance it will fail when we test it even without
the race detector.

<li> Workers will sometimes need to wait, e.g. reduces can't start
until the last map has finished. One possibility is for workers to
periodically ask the coordinator for work, sleeping
with <tt>time.Sleep()</tt> between each request. Another possibility
is for the relevant RPC handler in the coordinator to have a loop that
waits, either with <tt>time.Sleep()</tt> or <tt>sync.Cond</tt>. Go
runs the handler for each RPC in its own thread, so the fact that one
handler is waiting needn't prevent the coordinator from processing other
RPCs.

<li> The coordinator can't reliably distinguish between crashed workers,
workers that are alive but have stalled for some reason,
and workers that are executing but too slowly to be useful.
The best you can do is have the coordinator wait for
some amount of time, and then give up and re-issue the task to
a different worker. For this lab, have the coordinator wait for
ten seconds; after that the coordinator should assume the worker has
died (of course, it might not have).

<li> If you choose to implement Backup Tasks (Section 3.6), note that we test that your code doesn't
    schedule extraneous tasks when workers execute tasks without crashing.  Backup tasks should only
    be scheduled after some relatively long period of time (e.g., 10s).

<li> To test crash recovery, you can use the <tt>mrapps/crash.go</tt>
application plugin. It randomly exits in the Map and Reduce functions.

<li> To ensure that nobody observes partially written files in the presence of
    crashes, the MapReduce paper mentions the trick of using a temporary file
    and atomically renaming it once it is completely written. You can use
    <tt>ioutil.TempFile</tt> (or <tt>os.CreateTemp</tt> if you are running
    Go 1.17 or later) to create a temporary file and <tt>os.Rename</tt>
    to atomically rename it.

<li> <tt>test-mr.sh</tt> runs all its processes in the sub-directory <tt>mr-tmp</tt>, so if
    something goes wrong and you want to look at intermediate or output files, look there. Feel free to
    temporarily modify <tt>test-mr.sh</tt> to <tt>exit</tt> after the failing test, so the script does not
    continue testing (and overwrite the output files).

<li> <tt>test-mr-many.sh</tt> runs <tt>test-mr.sh</tt> many times in a row,
    which you may want to do in order to spot low-probability bugs.
    It takes as an argument the number of times to run
    the tests. You should not run several <tt>test-mr.sh</tt> instances in parallel because the
    coordinator will reuse the same socket, causing conflicts.

<li>Go RPC sends only struct fields whose names start with capital letters.
  Sub-structures must also have capitalized field names.

<li> When calling the RPC <tt>call()</tt> function, the
reply struct should contain all default values. RPC calls
should look like this:
<pre>
  reply := SomeType{}
  call(..., &reply)
</pre>
without setting any fields of reply before the call. If you
pass reply structures that have non-default fields, the RPC
system may silently return incorrect values.

</ul>

<h3>No-credit challenge exercises</h3>

<p class="challenge">
Implement your own MapReduce application (see examples in <tt>mrapps/*</tt>), e.g., Distributed
Grep (Section 2.3 of the MapReduce paper).

<p class="challenge">
Get your MapReduce coordinator and workers to run on separate machines, as they would in practice.
You will need to set up your RPCs to communicate over TCP/IP instead of Unix sockets (see
the commented out line in <tt>Coordinator.server()</tt>), and read/write files using a shared file
system. For example, you can <tt>ssh</tt> into multiple 
<a href=http://kb.mit.edu/confluence/display/istcontrib/Getting+Started+with+Athena>Athena cluster</a>
machines at MIT, which use 
<a href=http://kb.mit.edu/confluence/display/istcontrib/AFS+at+MIT+-+An+Introduction>AFS</a>
to share files; or you could rent a couple AWS instances and use 
<a href=https://aws.amazon.com/s3/>S3</a> for storage.

</body>
</html>

<!--  LocalWords:  Paxos Sharded shard sharding sharded Put's src shardcoordinator
-->
<!--  LocalWords:  shardkv cd TestUnreliable Go's RPCs RPC's GID px
-->
<!--  LocalWords:  kvpaxos Config ErrWrongGroup Handin gzipped czvf whoami tgz
-->
