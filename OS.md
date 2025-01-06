# Create RAMFS filesystem (memory mapped file)

```sh
sudo mount ramfs -t ramfs /opt/obrc_ramfs
```

# Load file into RAM

```sh
cp /opt/obrc/* /opt/obrc_ramfs
```
