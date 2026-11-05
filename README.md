# event-inbox-node

A Node server that takes a device reading.

The same reading can arrive more than once. A network failure makes the sender try again. The goal is to take the retry and still apply that reading once.
