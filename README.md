# piping stdout and stdout to different processes

OK. this is a rad trick from [bash-hackers](https://wiki.bash-hackers.org/howto/redirection_tutorial).

In brief, it is this:

```
{
  {
    cmd1 3>&- |
      cmd2 2>&3 3>&-
  } 2>&1 >&4 4>&- |
    cmd3 3>&- 4>&-
} 3>&2 4>&1
```

This opens additional file descriptors (3 & 4) for the first command so that the stdout can be piped to the stdin of one process, while the stderr is piped to the stdin of another.

The repo demos this. To run the demo for yourself:

```
$ ./redirect_demo
```

and witness output similar to:

```
FakeError: Bad thing with input: 13
FakeError: Bad thing with input: 20
FakeError: Bad thing with input: 25
Key:   FOO
Value: 10
------
Key:   BAR
Value: 8
------
Key:   BAZ
Value: 4
------
```


To explain this contrived example, there are scripts:

- mkmsgs ; generates data and errors
- summarize ; processes "good" messages (stdout)
- handle_err ; processes errors (stderr)

`mkmsgs` runs a short loop to either produce a tab-delimited line or an error message at a rate of 20%. Error messages are JSON, an intentionally different format.

`summarize` wants to make a short report based on the first field from the tab-delimited input.

`handle_err` expects the JSON input, and rewrites it as a human readable message.

These transform tasks serve no purpose other than to do "something" with the input.  

-----

# component demos

## summarize demo

```
$ ./mkmsgs 2>/dev/null
BAR     29567           1630080433670904414
BAZ     5971            1630080433674212256
BAZ     9452            1630080433677457675
FOO     24733           1630080433680776185
FOO     20813           1630080433684144240
BAR     32071           1630080433687428208
BAZ     32205           1630080433690910245
BAR     1105            1630080433694765168
BAR     32251           1630080433698829750
BAR     1952            1630080433702921884
BAR     4894            1630080433706170565
BAR     21713           1630080433709528545
FOO     1503            1630080433712744305
BAZ     12671           1630080433716023229
BAZ     15159           1630080433719520863
BAR     10594           1630080433722921689
FOO     30229           1630080433726215716
FOO     22071           1630080433729597691
BAZ     2657            1630080433732910249
BAZ     4418            1630080433736214261

$ ./mkmsgs 2>/dev/null | ./summarize
Key:   FOO
Value: 5
------
Key:   BAR
Value: 5
------
Key:   BAZ
Value: 10
------
```

## handle_err demo

```
$ ./mkmsgs 2>&1 >/dev/null
{"error":"Bad thing", "input":6}
{"error":"Bad thing", "input":13}
{"error":"Bad thing", "input":17}
{"error":"Bad thing", "input":21}
{"error":"Bad thing", "input":25}


$ ./mkmsgs 2>&1 >/dev/null | ./handle_err
FakeError: Bad thing with input: 2
FakeError: Bad thing with input: 10
FakeError: Bad thing with input: 11
FakeError: Bad thing with input: 22
```


## combined
```
$ redirects()
{
  {
    {
      $1 3>&- |
        $2 2>&3 3>&-
    } 2>&1 >&4 4>&- |
      $3 3>&- 4>&-
  } 3>&2 4>&1
}


$ redirects ./mkmsgs ./summarize ./handle_err
FakeError: Bad thing with input: 1
FakeError: Bad thing with input: 12
Key:   FOO
Value: 8
------
Key:   BAR
Value: 7
------
Key:   BAZ
Value: 8
------

```
