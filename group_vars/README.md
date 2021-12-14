
# Group Variables

## Hostgroup vars

None of these are necessary. Probabaly you won't even use these normally.

```
./base
./server
./headless
./desktop
```


## Actual usage

Most likeky you'll set the hostgroup in the inventory file.

`inventory/somegroupofhosts`
```
[control]
node1

[agents]
node2
node3

[control:vars]
hostgroup=server

[agents:vars]
hostgroup=headless
```

If you need to, you can set it in `group_vars` or `host_vars`.
