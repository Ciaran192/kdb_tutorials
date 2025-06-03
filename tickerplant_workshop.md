# How to set up a tickerplant

This is a step by step guide that will run through how to set up a tickerplant streaming system for practice on your own computer. We will get answers to:

1. How do I to start a tickerplant?
2. How do I start a basic subscriber?
3. How do I create a feedhandler?
4. How do I set up a HDB and RDB?
5. What does the EOD process do?

## First, let's get set up

To follow along with this, you will need:

- q installed (you can read about how to do that [here](https://code.kx.com/q/learn/install/))
- This tutorial will use bash. If you are not working on Mac or Linux, you can run Linux on your Windows machine using [Windows Subsystem for Linux](https://learn.microsoft.com/en-us/Windows/wsl/install).
- If you are running q using wsl, make sure to download and install q for Linux.

Next, you will need to download the [tickerplant scripts.](https://github.com/KxSystems/kdb-tick) Keep these in the same folder structure as they are in the repository.

Open a bash terminal, and navigate into the folder where your tick.q file is located.

```
$ cd <tick scripts root>/
$ ls
tick  tick.q
```

Finally, in order to run a tickerplant, we need to have a schema file in our `tick/` directory. this is a q file that we can call whatever we want. We can just use the [sample](https://code.kx.com/q/architecture/tickq/#schema-file) from the KX website in this tutorial.

Create a file `<tick scripts root>/tick/schema.q`:

```
$ touch tick/schema.q
```

and copy the code you find [here](https://code.kx.com/q/architecture/tickq/#schema-file) into the file.

## How to set up a tickerplant

in the same directory as tick.q, run:

`$ q tick.q -q schema logs`

This says:

*Start an interactive q session running on port 5010 (default tickerplant port) with `tick/schema.q` as my schema file, and store my tickerplant logs under `./logs/`*

You can inspect all the special tickerplant variables and functions by looking at the `.u` namespace in this session.

## How to set up a basic subscriber

A subscriber needs to do three things:

1. Open a connection to the tickerplant.
2. Call `.u.sub[<tables>;<syms of interest>]` on the tickerplant, and use the response to define empty tables in its own session.
3. Define a upd function, since that is what the tickerplant calls on its subcribers. ([Here](https://github.com/KxSystems/kdb-tick/blob/85c08ff192b0a103b323246c7300a37919be6159/tick.q#L39) is where you can see how it does it in batch mode, [here](https://github.com/KxSystems/kdb-tick/blob/85c08ff192b0a103b323246c7300a37919be6159/tick.q#L45) is the logic for how it does it in real time mode.)

So open a new q session.

`$ q`

and run:

```
q)h:hopen 5010
q)msg:h (`.u.sub;`;`)
q).[set] each msg
q)upd:insert
```

## How to set up a mock feedhandler

This is not a *real* feedhandler, so to speak. It's just a q process which is going to simulate trades and quotes being sent to our tickerplant. 

Real feedhandlers connect to data source systems and serialize the data into q format to be sent to the tickerplant. They can be written in many languages. You can see links to different interfaces for writing feedhandlers in different languages on the KX website. For example, you can see the documentation for c/c++ [here](https://code.kx.com/q/interfaces/c-client-for-q/).

So for our feedhandler, start up a q process.

`$ q`

**In feehandler run:**
```
q)h:hopen 5010
q)createTrade:{raze (1?.z.n;1?`AAPL`GOOG`IBM;1?10;1?10f;1?`n`t;1?`b`s)}
q)createQuote:{raze (1?.z.n;1?`AAPL`GOOG`IBM;1?10f;1?10f)}
q).z.ts:{h (`.u.upd;`trade;createTrade[]);h (`.u.upd;`quote;createQuote[]);}
q)\t 1000
```

Now our feedhandler should start sending updates to our tickerplant once per second. The tickerplant will subsequently send updates downstream to the real time subscriber.

If you look at the trade table in the subscriber, you should see something like:
```
time                 sym  size price    exchange side
-----------------------------------------------------
0D10:21:19.453834262 GOOG 3    8.671096 n        b
0D04:53:50.715620244 AAPL 8    6.789082 n        s
0D05:28:53.609647936 GOOG 8    9.149882 t        s
0D07:01:37.596679405 GOOG 9    9.216436 n        b
0D02:28:48.962298638 GOOG 5    3.839461 n        s
0D00:06:07.591199564 IBM  4    1.53227  n        s
0D05:39:44.061917804 IBM  4    5.823059 t        s
0D07:53:17.764974813 GOOG 9    7.67486  n        s
0D12:39:37.720492799 AAPL 1    4.448492 n        s
0D08:04:39.468309452 AAPL 1    3.573039 n        s
0D01:45:48.357382059 IBM  7    4.101914 t        b
...
```

Which means that everything is working properly.

### Bonus: How to create and identify a slow subscriber

On your subscriber process, enable the [error trap](https://code.kx.com/q/basics/syscmds/#e-error-trap-clients) for client requests.

**In subscriber:**
`q)\e 1`

This will cause the process to suspend execution if it receives any client requests that cause an error.

Next, set the `upd` function to be something that will cause an error.

**In subscriber:**
``q)upd:{[t;x] `e + x}``

Now, our subscriber will suspend execution.

**In subscriber:**
````
q)'type
  [0]  upd:{[t;x] `e + x}
                     ^
q))
````

Messages that the tickerplant sends to this subscriber will no longer be processed. So where do they end up?
Well first, they get stored in the receiver TCP window. Once this is full, messages will be queued in the tickerplant. the tickerplants memory will fill up until it crashes with a 'wsfull error, or until it takes up all the memory on your machine and crashes the machine itself. 

If we look at the tickerplant and take a look at [.z.W](https://code.kx.com/q/ref/dotz/#zw-handles), we won't see anything too concerning just yet.

**In TP:**
```
q).z.W
7| 0
8| 0
```

So, on our feedhandler, let's turn up the heat by changing the number of updates from once every second to once every 3ms.

**In feedhandler:**
```
q)\t 3
```

Now, if we wait a few seconds, we will see the byte queue associated with the handle for our RTS really start to fill up on our TP.

**In TP:**
```
q).z.W
7| 2957910
8| 0
q).z.W
7| 4285522
8| 0
```

We can also observe the amount of memory usage increasing in the output of `htop`.

So assuming we say that a queue of 1GB means enough is enough, we can run the following on our TP:

**In TP:**
```
q)slowSubscriberHandle:first where .z.W > 1000000
q).u.del[;slowSubscriberHandle] each .u.t
q)hclose slowSubscriberHandle
```

The crisis is averted.

## The full picture: How to set up an RDB and a HDB

**First**, we'll set up a HDB.

The vanilla RDB expects the HDB to be running in a directory underneath where the TP logs are stored. It expects the name of that HDB directory to be the name of our schema file. 

So for example, our schema file is called `schema.q`. So our HDB will be stored underneath `<tick scripts root>/logs/schema/`.

Run:

```
$ cd  <tick scripts root>/logs/
$ mkdir schema
$ cd schema
$ q -p 5012
```

**Secondly**, we'll set up our RDB. In our root, run:

```
$ q tick/r.q localhost:5010 localhost:5012
```

The two arguments are the ports of our TP and HDB processes respectively. We didn't need to include this in our case, since 5010 and 5012 are the default ports for the TP and HDB in the r.q script. We could just as well have run:

```
$ q tick/r.q
```

If you have completed this tutorial from the beginning, you might notice that the RDB takes some time to load when you initially set it up. This is because it is replaying the tickerplant log file [using](https://github.com/KxSystems/kdb-tick/blob/85c08ff192b0a103b323246c7300a37919be6159/tick/r.q#L15) `.u.rep`. You will notice that once it finishes, it has all the updates for our trade and quote tables.

That's the full setup! Finally, let's take a look at the end of day process.

## End of day processing

Typically, our end of day process is initiated by the tickerplant once it notices that a new day has begun. It runs its `.u.endofday` function to send a message to each of its subscribers to initiate their `.u.end` functions. It's in the vanilla `.u.end` function where the end of day writedown procedure is kicked off. 

We can see how this happens by navigating to our RDB, and running:

**In RDB:**
```
q).u.end[.z.d]
```

This will use [.Q.hdpf](https://code.kx.com/q/ref/dotq/#hdpf-save-tables) to write down the tables, clear out its own tables, and tell the HDB to reload the partitioned database under `<tick scripts root>/logs/schema/` using `\l .`.

If this all worked the way it was supposed to, your trade and quote table will be saved as splayed tables under `logs/schema/<date>/trade/` and `logs/schema/<date>/quote/`. If you inspect the HDB process, you will see that the trade and quote tables are queryable there, with the extra virtual column date.

**In HDB:**
```
q)select from trade where date=.z.d
date       sym  time                 size price      exchange side
------------------------------------------------------------------
2025.06.03 AAPL 0D04:53:50.715620244 8    6.789082   n        s
2025.06.03 AAPL 0D12:39:37.720492799 1    4.448492   n        s
2025.06.03 AAPL 0D08:04:39.468309452 1    3.573039   n        s
2025.06.03 AAPL 0D03:53:52.922470163 6    8.665565   n        b
2025.06.03 AAPL 0D01:38:11.572269153 3    8.649262   n        b
2025.06.03 AAPL 0D00:58:57.620736022 1    8.639591   n        b
2025.06.03 AAPL 0D04:19:34.266400762 0    3.338806   t        s
2025.06.03 AAPL 0D02:21:35.909395519 9    3.813679   t        s
..
```

## Conclusion

This tutorial shows you how to implement a vanilla tickerplant setup. It should serve as a useful entrypoint to begin playing with kdb+ streaming and start creating your own systems.

Ciarán Daly, June 3rd 2025.
