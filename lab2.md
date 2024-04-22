
<!DOCTYPE html>
<html>
<head>
<link rel="stylesheet" href="../style.css" type="text/css">
</head>
<body>
<div align="center">
<h2><a href="../index.html">6.5840</a> - Spring 2024</h2>
<h1>6.5840 Lab 2: Key/Value Server</h1>

<p>
  <b><a href="collab.html">Collaboration policy</a></b> //
  <b><a href="submit.html">Submit lab</a></b> //
  <b><a href="go.html">Setup Go</a></b> //
  <b><a href="guidance.html">Guidance</a></b> //
  <b><a href="https://piazza.com/mit/spring2024/65840">Piazza</a></b>
</p>

</div>

<hr>

<h3>Introduction</h3>

<p>
In this lab you will build a key/value server for a single machine
that ensures that each operation is executed exactly once despite
network failures and that the operations
are <a href="../papers/linearizability-faq.txt">linearizable</a>. Later
labs will replicate a server like this one to handle server crashes.

<p>
Clients can send three different RPCs to the key/value server:
<tt>Put(key, value)</tt>,
<tt>Append(key, arg)</tt>, and <tt>Get(key)</tt>. The server maintains 
an in-memory map of key/value pairs. Keys and values are strings.
<tt>Put(key, value)</tt> installs or replaces the value for a particular key in
the map, <tt>Append(key, arg)</tt> appends arg to key's value <i>and</i>
returns the old value, and <tt>Get(key)</tt> fetches the current value
for the key. A <tt>Get</tt> for a non-existent key should return an
empty string. An <tt>Append</tt> to a non-existent key should act
as if the existing value were a zero-length string.
Each client talks to the server through
a <tt>Clerk</tt> with Put/Append/Get methods. A <tt>Clerk</tt> manages
RPC interactions with the server.

<p>Your server must arrange that application calls
  to <tt>Clerk</tt> Get/Put/Append methods be linearizable. If
client requests aren't concurrent,
each client Get/Put/Append call should observe
the modifications to the state implied by the preceding sequence of
calls. For concurrent calls, the return values and final state must be
the same as if the operations had executed one at a time in some
order. Calls are concurrent if they overlap in time: for example, if
client X calls <tt>Clerk.Put()</tt>, and client Y
calls <tt>Clerk.Append()</tt>, and then client X's call returns. A
call must observe the effects of all calls that have completed before
the call starts.

<p>
Linearizability is convenient for applications because it's the
behavior you'd see from a single server that processes requests one at
a time. For example, if one client gets a successful response from the
server for an update request, subsequently launched reads from other
clients are guaranteed to see the effects of that update. Providing
linearizability is relatively easy for a single server.

<h3>Getting Started</h3>

<p>
We supply you with skeleton code and tests in <tt>src/kvsrv</tt>. You will
need to modify <tt>kvsrv/client.go</tt>, <tt>kvsrv/server.go</tt>, and
<tt>kvsrv/common.go</tt>.

<p>
To get up and running, execute the following commands.
Don't forget the <tt>git pull</tt> to get the latest software.
<pre>
$ cd ~/6.5840
$ git pull
...
$ cd src/kvsrv
$ go test
...
$</pre>

<h3>Key/value server with no network failures <script>g("easy")</script></h3>

<div class="todo">
<p>
Your first task is to implement a solution that works when there are no dropped
messages.

<p>
You'll need to add RPC-sending code to the Clerk Put/Append/Get
methods in <tt>client.go</tt>, and implement
<tt>Put</tt>, <tt>Append()</tt> and <tt>Get()</tt> RPC handlers in
<tt>server.go</tt>.

<p>
You have completed this task when you
pass the first two tests in the
test suite: "one client" and "many clients".
</div>

<ul class="hints">

<li>
Check that your code is race-free using
<tt>go test -race</tt>.

</ul>

<h3>Key/value server with dropped messages <script>g("easy")</script></h3>

Now you should modify your solution to continue in the face of dropped
messages (e.g., RPC requests and RPC replies).
If a message was lost, then the client's <tt>ck.server.Call()</tt>
will return <tt>false</tt> (more precisely, <tt>Call()</tt> waits
for a reply message for a timeout interval, and returns false
if no reply arrives within that time).
One problem you'll face is that a
<tt>Clerk</tt> may have to send an RPC multiple times until it
succeeds. Each call to
<tt>Clerk.Put()</tt> or <tt>Clerk.Append()</tt>, however, should
result in just a <i>single</i> execution, so you will have to ensure
that the re-send doesn't result in the server executing
the request twice.

<div class="todo">
<p> Add code to <tt>Clerk</tt> to retry if doesn't receive a reply,
and to
  <tt>server.go</tt> to filter duplicates if the operation requires
it. These notes include guidance
on <a href="../notes/l-raft-QA.txt">duplicate detection</a>.
</div>

<ul class="hints">

<li>
You will need to uniquely identify client operations to ensure that
the key/value server executes each one just once.

<li>You will have to think carefully about what state the server must
  maintain for handling duplicate <tt>Get()</tt>, <tt>Put()</tt>,
  and <tt>Append()</tt> requests, if any at all.


<li>
Your scheme for duplicate detection should free server memory quickly,
for example by having each RPC imply that the client has seen the
reply for its previous RPC. It's OK to assume that a client will make
only one call into a Clerk at a time.

</ul>

<p>
Your code should now pass all tests, like this:

<pre>
$ go test
Test: one client ...
  ... Passed -- t  3.8 nrpc 31135 ops 31135
Test: many clients ...
  ... Passed -- t  4.7 nrpc 102853 ops 102853
Test: unreliable net, many clients ...
  ... Passed -- t  4.1 nrpc   580 ops  496
Test: concurrent append to same key, unreliable ...
  ... Passed -- t  0.6 nrpc    61 ops   52
Test: memory use get ...
  ... Passed -- t  0.4 nrpc     4 ops    0
Test: memory use put ...
  ... Passed -- t  0.2 nrpc     2 ops    0
Test: memory use append ...
  ... Passed -- t  0.4 nrpc     2 ops    0
Test: memory use many puts ...
  ... Passed -- t 11.5 nrpc 100000 ops    0
Test: memory use many gets ...
  ... Passed -- t 12.2 nrpc 100001 ops    0
PASS
ok      6.5840/kvsrv    39.000s
</pre>

<p>
The numbers after each <tt>Passed</tt> are real time in seconds,
number of RPCs sent (including client RPCs), and
number of key/value operations executed (<tt>Clerk</tt> Get/Put/Append
calls).


</body>
</html>
