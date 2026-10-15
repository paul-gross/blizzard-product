# Hub

The daemon that is the fleet's shared memory and front door, built colocated, moved off-machine, then packaged for
anyone to run — each epic frozen as it shipped.

| Where                                | Read when                                                                                              |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| [initiative.md](./initiative.md)     | Reasoning about the hub as a whole — why where it runs is a deployment choice, and the order it landed |
| [separation.md](./separation.md)     | Looking up the colocated hub, the runner↔hub protocol, and what the hub owns                           |
| [remote.md](./remote.md)             | Looking up the off-machine hub every runner signs in to over the network                               |
| [distribution.md](./distribution.md) | Looking up the public hub image and the release discipline behind it                                   |
