# Grafana

```bash
# filesystem usage bar gauge
100 * (node_filesystem_size_bytes{instance=$instance, fstype!~tmp.*} - node_filesystem_free_bytes{instance=$instance, fstype!~tmp.*}) / (node_filesystem_size_bytes{instance=$instance, fstype!~tmp.*})
```

```bash
# stats
max by(pretty_name)(node_os_info{instance=$instance})
# transform data -> organize fields by name
# 
```

