# Reproduction of the effect on push connection of a blocked pull connection

`pull` connections in RTT were meant to avoid affecting a running system when
using monitoring tools (e.g. rock-display, syskit ide) since the transmission
was initiated by the monitoring side. However, it turned out that this isolation
was broken because of RTT's "port signalling" feature, that is required for port
data callbacks such as port driven tasks.

The issue was that on push (when the component writes a new data on the port),
pull connections would do a remote call to signify that new data was available,
instead of sending the sample itself (in the push case).

As a first attempt to fix this, we changed the `remoteSignal` CORBA call to be
`oneway`. Testing seemed to show that it indeed fixed the problem. However, we
noticed even after the introduction of `oneway` that the problem persisted.
Systems would start to massively misbehave if connection to a monitoring too was
interrupted in a non-clean fashion (e.g. comm link broken, laptop went to sleep)

## Reproducing the blocking behaviour

The test setup is made of four scripts:
- source: the data source. Run it first. It displays its PID.
- pull_sink: periodic reader of the data from the source configured with `pull:
  true`, using both port-to-port connection and data reader. The first field
  of each line is the PID.
- push_sink: periodic reader of the data from the source configured with `pull:
  false`, using both port-to-port connection and data reader. The first field
  of each line is the PID.
- block: script that simplifies setting up iptables rules to block TCP
  connections

Start the first three scripts and make sure the two sink scripts display data
being received. Then run the following command, where PULL_SINK_PID is the PID
of the `pull_sink` process (first column of the displayed lines)

```
sudo ss -tpn | grep $PULL_SINK_PID
```

This would for instance output something like:

```
ESTAB      0      0                    127.0.0.1:42712                127.0.0.1:2809
ESTAB      0      0                192.168.68.63:50482            192.168.68.63:51691
ESTAB      68     0       [::ffff:192.168.68.63]:43833   [::ffff:192.168.68.63]:45746
ESTAB      0      0       [::ffff:192.168.68.63]:43833   [::ffff:192.168.68.63]:45750
```

Then run

```
sudo ./block $IP $PORT0 $PORT1
```

Where `IP` is the IP of the source displayed by `ss` and `PORT*` the ports
displayed by `ss`, on the first column. In the example above, we would run

```
sudo ./block 192.168.68.63 50482 43833
```

Go check if the `pull_sink` output has frozen. It should.

Then wait a bit, the `push_sink` scripts should stop as well after a while. You
can monitor the progress of what happens by running the following, replacing the
PORT0|PORT1|... by the actual ports from before:

```
watch sh -c "ss -tpn | egrep '$PORT0|$PORT1'"
```

This will display a socket whose send queue is being filled. This is the
connection from the source to `pull_sink`, which is now blocked. Because of
`oneway`, the calls do return and the RTT send thread can continue. However,
once the queue is full, omniORB blocks.

## Validating the `signalling` flag as a solution

What we implemented to solve this is the addition of a new flag on connection
policies, the `signalling` flag. It disables signalling on pull connections.

Run the same test, replacing `pull_sink` by `pull_sink_no_signalling` and
validate that the send queue is not being filled anymore, and that the
`push_sink` does not block.

