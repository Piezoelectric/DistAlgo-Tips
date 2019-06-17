# DistAlgo-Tips
Tips and observations about the programming language DistAlgo

Since DistAlgo doesn't have official documentation I'm going to scribble some observations here. It's not very organized, but I hope some of the content here is helpful to someone.

## Installing

`python3.6 -m pip install --user pyDistAlgo`

Using `pythonX.X -m pip` specifies which version of python to install a module to. Using `pip install --user pyDistAlgo` installs only for the current user, as opposed to the default (a system-wide install). You can combine the two as above.

This also works on the TAMU compute server. (Where you're not allowed to do a system-wide install anyway.)

## Running

Running with `python -m da sourcefile.da` is the most basic run possible. 

Running with `python -m da --help` or `python -m da -h` brings up help, as well as options for running DistAlgo.

Logging options:
 * `-f` or `--logfile` creates a log file for this run 
 * `--logfilename LOGFILENAME` specifies the log file's name
 * `--logfilelevel` specifies the level of info that gets written to the logfile

Example: `python -m da -f --logfilelevel info --logfilename sample.txt sample.da <dafile arguments>` (Yes, both -f and --logfilename should be specified, --logfilename alone is insufficient).

For resource limits:
 * `-I` or `--default-proc-impl` lets you choose the default DistAlgo process implementation. That is, you can choose whether to run every node using a process for each node, or run every node using a thread for each node. 
 
`-I` defaults to processes if unspecified. So if you're getting errors about OSProcessException, switch to threads instead.

So: `python3.6 -m da --default-proc-impl thread --logfile --logfilename FILENAME.txt --logfilelevel info <DAFILE>.da <dafile arguments>`

For more complex programs you may get some error `RecursionError('maximum recursion depth exceeded in comparison',) Fatal Python error: Cannot recover from stack overflow.` -- even if there is no recursion in your code. In this case, you'll want to import `sys` and add `sys.setrecursionlimit(10**6)` to your `main()` function. I personally have found this sufficient for working with the TAMU servers, but depending on what you do, you may need to increase the total stack size as well (see https://stackoverflow.com/questions/3323001/ ).

## Resources of interest

https://github.com/DistAlgo/distalgo -- Official repo for DistAlgo's python implementation

https://sites.google.com/site/distalgo/tutorial -- Homepage for DistAlgo

https://drive.google.com/file/d/0B0MWH8ngLAIFTzFna1lOTTF4UlE/view -- language specification PDF (available from distalgo site, click "Language Description"

https://github.com/DistAlgo/distalgo/tree/master/da/examples -- Examples.

## Under the hood -- the Python implementation of the DistAlgo language

All user-made classes are subclasses of `DistProcess`, and implement a version of `setup` and `run`.

When actually running DistAlgo code, a ProcessContainer instance runs a DistAlgo process instance. There are two implementations of ProcessContainer: one which uses Processes and the multiprocessing library, and one which uses Threads and the threading library. So you have a choice: you can choose to run every node using processes, or run every node using threads.

https://github.com/DistAlgo/distalgo/blob/master/da/sim.py -- several interesting things here.
 * The definition of DistAlgo's `DistProcess` class is here. 
 * `DistProcess`'s other builtin methods are listed here. (The builtin methods are also accessible by running `python -m da -B`.)
 * You can look up runtime error messages here. For example, in `DistProcess.__process_event()` there's the line `self._log.error("Exception while processing message %r: %r", message, e)`.
 * This is also where the `ProcessContainer` class and subclasses are defined. 

https://github.com/DistAlgo/distalgo/blob/master/da/common.py -- This is where DistAlgo's `ProcessId` class is defined. A little more on this below.

https://github.com/DistAlgo/distalgo/blob/master/da/compiler/parser.py -- The DistAlgo compiler + parser. (You can look up compilation error messages here.)

## Can't access attributes of a class

Unlike Python, DistAlgo doesn't let you directly access attributes of a class. For example:

```python
class Pong(process):
    self.attribute = 0

    def setup(total_pings:int): 
        self.attribute = 1

# [...]

def main():
    pong = new(Pong, [nrounds * npings], num= 1)
    setup(ping, (pong, nrounds))
    print(type(pong))
    for p in pong:
        print(type(p))
        print(p.attribute) # <-- Pay attention to this line
```

Will fail with error `AttributeError: 'ProcessId' object has no attribute 'attribute'`

This is because DistAlgo doesn't instantiate the class itself. Instead, it creates a new process, and gives you a handle for that process -- that's what the ProcessId object is. ProcessId is a class built into DistAlgo, so you can't really modify it. 

This can be demonstrated by modifying one of the DA examples (below is a modified version of `ping.da`):

```python
import sys

# This example has been modified to demonstrate the concept
# of Distalgo ProcessIDs

class Pong(process):
    self.attribute = 0

    def setup(total_pings:int): 
        self.attribute = 1

    def run():
        await(total_pings == 0)

    def receive(msg=('Ping',), from_=p):
        output("Pinged")
        send(('Pong',), to=p)
        total_pings -= 1

class Ping(process):
    def setup(p:Pong, nrounds:int): pass

    def run():
        for i in range(nrounds):
            clk = logical_clock()
            send(('Ping',), to=p)
            await(some(received(('Pong',), clk=rclk), has=(rclk > clk)))

    def receive(msg=('Pong',)):
        output("Ponged.")

def main():
    nrounds = int(sys.argv[1]) if len(sys.argv) > 1 else 3
    npings = int(sys.argv[2]) if len(sys.argv) > 2 else 3
    config(clock='Lamport')
    pong = new(Pong, [nrounds * npings], num= 1)
    ping = new(Ping, num= npings)
    setup(ping, (pong, nrounds))
    print(type(pong))
    for p in pong:
        print(type(p))
        print(p.attribute) #Comment this line to get program to run
    start(pong)
    start(ping)

```

(I don't know what it looks like when you run this with `--default-proc-impl thread`. You could get a ThreadID object instead, or something. But in any case, I imagine you still can't access an object's data attributes directly.)

## Non-usage of "Self" keyword

Recalling the earlier PingPong example, notice that I stuck `self.attribute` in the class definition, even before `setup()`. 

I don't actually know if the "self.attribute" is necessary for all things. Consider 2pcommit from the examples (https://github.com/DistAlgo/distalgo/blob/master/da/examples/2pcommit/orig.da):

```python
class Cohort(process):
    def setup(failure_rate):
    	self.terminate = False
        
    [...]
    
    def abort(): output('abort'); terminate = True
```

Notice that the `setup()` call has `self.terminate` but the `abort()` call only has `terminate`, no `self`. I'm inclined to believe that "self" is only required in the `setup()` call, and every other function implicitly binds "self".

In any case, the `self` keyword definitely doesn't work *exactly* the same way as it does in Python. (This is something I should ask about.)

## Asynchronous Non-Blocking Waiting

Suppose you want a node to wait for some time before doing X, but while it's waiting, you still want it to respond to external messages. `time.sleep()` won't do what you need. `time.sleep` will block the thread entirely, so it will not respond to any messages until the `time.sleep()` duration is over. Use `if await(False) : pass ; elif timeout(duration): pass` syntax, as seen in the example below.

Take the following code as a trivial example:

```python
import time

class Node(process):
    def setup(id:int, waitTime:int):
        pass
        
    def stall():
        if await(False): pass
        elif timeout(waitTime): pass
    
    def run():
        stall()
        output("Node", id, "finishes after", waitTime, "seconds")
        
    def receive (msg=('test',)):
        output("Response to test from", id, "successful")

def main():
    nodes = []
    for i in range(10):
        n = new(Node)
        nodes.append(n)
        setup(n, (i, i))
    for n in nodes:
        start(n)
    #The important question: Do sleeping nodes respond to messages?
    for n in nodes:
        send(('test',), to=n)
```

The desired output is:

```
[2293] blockingTest5.Node<Node:63002>:OUTPUT: Node 0 finishes after 0 seconds
[2043] blockingTest5.Node<Node:63003>:OUTPUT: Response to test from 1 successful
[1591] blockingTest5.Node<Node:63005>:OUTPUT: Response to test from 3 successful
[1123] blockingTest5.Node<Node:63007>:OUTPUT: Response to test from 5 successful
[109] blockingTest5.Node<Node:6300b>:OUTPUT: Response to test from 9 successful
[1341] blockingTest5.Node<Node:63006>:OUTPUT: Response to test from 4 successful
[358] blockingTest5.Node<Node:6300a>:OUTPUT: Response to test from 8 successful
[655] blockingTest5.Node<Node:63009>:OUTPUT: Response to test from 7 successful
[1840] blockingTest5.Node<Node:63004>:OUTPUT: Response to test from 2 successful
[889] blockingTest5.Node<Node:63008>:OUTPUT: Response to test from 6 successful
[3042] blockingTest5.Node<Node:63003>:OUTPUT: Node 1 finishes after 1 seconds
[3822] blockingTest5.Node<Node:63004>:OUTPUT: Node 2 finishes after 2 seconds
[4602] blockingTest5.Node<Node:63005>:OUTPUT: Node 3 finishes after 3 seconds
[5335] blockingTest5.Node<Node:63006>:OUTPUT: Node 4 finishes after 4 seconds
[6115] blockingTest5.Node<Node:63007>:OUTPUT: Node 5 finishes after 5 seconds
[6864] blockingTest5.Node<Node:63008>:OUTPUT: Node 6 finishes after 6 seconds
[7628] blockingTest5.Node<Node:63009>:OUTPUT: Node 7 finishes after 7 seconds
[8330] blockingTest5.Node<Node:6300a>:OUTPUT: Node 8 finishes after 8 seconds
[9094] blockingTest5.Node<Node:6300b>:OUTPUT: Node 9 finishes after 9 seconds
```

Replace the `stall()` method with a call to `time.sleep()`, and one finds the following output:

```
[2293] blockingTest5.Node<Node:63002>:OUTPUT: Node 0 finishes after 0 seconds
[3042] blockingTest5.Node<Node:63003>:OUTPUT: Node 1 finishes after 1 seconds
[3822] blockingTest5.Node<Node:63004>:OUTPUT: Node 2 finishes after 2 seconds
[4602] blockingTest5.Node<Node:63005>:OUTPUT: Node 3 finishes after 3 seconds
[5335] blockingTest5.Node<Node:63006>:OUTPUT: Node 4 finishes after 4 seconds
[6115] blockingTest5.Node<Node:63007>:OUTPUT: Node 5 finishes after 5 seconds
[6864] blockingTest5.Node<Node:63008>:OUTPUT: Node 6 finishes after 6 seconds
[7628] blockingTest5.Node<Node:63009>:OUTPUT: Node 7 finishes after 7 seconds
[8330] blockingTest5.Node<Node:6300a>:OUTPUT: Node 8 finishes after 8 seconds
[9094] blockingTest5.Node<Node:6300b>:OUTPUT: Node 9 finishes after 9 seconds
```
No node responds to `test` messages.

## Synchronous and Asynchronous Message Passing

For synchronous algorithms, using the `await(x)` handler is ideal -- code will not progress beyond the `await(x)` statement until x is true. In particular, you may want `await(received(msg = 'foo'))`; `received()` will become true once the node has received a message 'foo'. 

I haven't done a whole lot of work with synchronous algorithms, so this section is WIP.

For asynchronous algorithms, don't use `await(x)`, since that enforces some synchrony/waiting. Instead, do a combination of things:

* Define all messages you want to respond to with `receive()` handlers.
* When receiving messages, add some "message delay" using the `if await(False): pass ; elif(timeout): pass` method outlined above. 
* In your process's `run()` method, put `await(False)` to keep the process running forever. Since it's running forever, it will "stay awake" to respond to messages, and messages it receives will trigger the appropriate `receive()` handlers. 

Note that it's better to have the random delays in the message *receival* code. Suppose the delay were on the *sender's* side, and we had a broadcasting system. Then, when broadcasting to multiple nodes, same delay amount would be applied to all recipients. This is not ideal asynchrony.

Also note that receive handlers are finicky in what they respond to:
```
receive(msg = 'abc') waits for a message "abc", string type
receive(msg = ('abc')) waits for a message "abc", string type
receive(msg = ('abc',)) waits for a message ('abc',) tuple type
```
(You can test the above with `type( ('abc',) )` in a python interpreter.)

Likewise, pay attention to how your messages are spelled.  If you have one node with a handler `receive(msg = "whatever")` and another node sending the message `"Whatever"`, the first node's handler will never activate. 

One more thing about receive handlers: In addition to defining receiving a message, you can also ask for metadata about the message, such as the clock and which node sent it. 

There's some weird syntax thing I'm really not sure about. For `received()` inside `await`, I've seen some code with:

```await(received(msg = 'whatever', from_=_p), ...) #don't run this, it's just a sketch```

But I've also seen code that looks like:

```await(received(msg = 'whatever', from_ =p), ...) #don't run this```

And for a `def receive()` handler on its own, I've written code that's like:

```def receive(msg = 'whatever', from_ = p)```

The reason the first underscore in `from_` is necessary is because "from" is a reserved keyword in Python. However I'm not sure why some code uses `from_=_` and some uses just `from_ =`. This is another thing to ask about. When I wrote `def receive(msg = 'whatever', from_=_p)` I got a warning "new variable '%s' introduced by bound pattern." (see parser.py) -- this will eventually lead to bad behavior.

