# Erlang Ring Benchmark

5 September 2019

I'm reading Joe Armstrong's Programming Erlang second edition (again). At the end of chapter 12 about concurrent programming there's an exercise to create a ring benchmark. This spawns a number of Erlang processes in a "ring", where each process passes a message to the next process in the ring, until it reaches the original sender of the message. We spawn a ring of N processes and send M messages around the ring, then we're supposed to write a blog about it and publish the results in the internet.

I wrote my solution as follows: we start with process N and let it spawn a next process to pass messages to, which will be process (N - 1). We repeat this until Process 0, in which case we do not spawn a new process but rather send the message back to the "master" process.
To send the messages around, we use a similar technique where we start with message M, and recursively send message (M - 1) until we reach message 0.
After we have passed all messages around, we stop the timer and report the amount of time that passed. Finally, we clean up the ring by sending the `done` message around the ring, which ends every process.

```erlang
-module(ring_benchmark).
-export([start/2]).

start(N, M) ->
    statistics(wall_clock),
    MasterPid = self(),
    Pid = spawn(fun() -> pass(MasterPid, N) end),
    send(Pid, M),
    wait(),
    {_, T} = statistics(wall_clock),
    Seconds = T / 1000,
    io:format("~p seconds~n", [Seconds]),
    Pid ! done.

%% wait until we receive message 1
wait() ->
    receive
        X when X == 1 ->
            void;
        _ ->
            wait()
    end.

%% spawn a new process to send messages to
%% then forward all messages to it
pass(MasterPid, N) ->
    case N of
         0 -> loop(MasterPid);
         _ -> Pid = spawn(fun() -> pass(MasterPid, N - 1) end),
              loop(Pid)
    end.

%% loop that passes every message to the next Pid
loop(Pid) ->
    receive
        done ->
            Pid ! done;
        M ->
            Pid ! M,
            loop(Pid)
    end.

%% send M messages around the ring starting at Pid
send(_, M) when M =< 0 -> void;
send(Pid, M) ->
    Pid ! M,
    send(Pid, M - 1).
```

Let's run the benchmark in an Erlang shell:

```
$ erl
Erlang/OTP 21 [erts-10.3.5.4] [source] [64-bit] [smp:8:8] [ds:8:8:10] [async-threads:1] [hipe]

Eshell V10.3.5.4  (abort with ^G)
1> c(ring_benchmark).
{ok,ring_benchmark}
2> ring_benchmark:start(10000, 10000).
4.404 seconds
done
3>
```

As you can see, it takes about 4.5 seconds to pass 10,000 messages along a ring of a 10,000 processes, which includes the time to start (spawn) all these processes.
Note: my laptop is running Linux (Fedora 30) and has an Intel Core i5-8250U processor with 16GB of RAM, running it on different hardware could yield different results.

Of course after writing this I looked for other blog posts describing this exercise, and one in particular caught my eye: [https://basicbitch.software/posts/2016-08-08-Ring-Benchmark-in-Erlang-and-Go.html](https://basicbitch.software/posts/2016-08-08-Ring-Benchmark-in-Erlang-and-Go.html)
The author compares Erlang with Go, which is very good as Joe Armstrong asked the user to implement the benchmark in a different language for comparison. The Go benchmark runs about 15% faster than the author's Erlang implementation. Of course, I was curious about the performance of Go solution on my own laptop, so I installed Go and tried it out:

```
$ go run ringbenchmark.go
Finished! Exiting...
Loop took 31.073469642s
```

Surprisingly, that is a lot slower than my own Erlang solution. Looking at CPU usage I can see that Erlang uses 100% of all 8 threads during the benchmark, while Go doesn't seem to be busy with CPU all that much.
Of course, this is just a silly benchmark, and it doesn't really mean anything, as there's probably not a real use case where passing messages around a ring is very imporant.
Personally I like how Erlang makes concurrency and parallellism easy with lightweight processes and message passing. I like dynamic typing and the functional aspects of the language. I will spend some more time going through the rest of the book and writing some small applications in Erlang.
