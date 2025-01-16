# Create RAMFS filesystem (memory mapped file)

```sh
sudo mount ramfs -t ramfs /opt/obrc_ramfs
```

# Load file into RAM

```sh
cp /opt/obrc/* /opt/obrc_ramfs
```

# Create `ramfs` location for Postgres `ramfs` tablespace

```sh
sudo mkdir /opt/obrc_ramfs/psql
sudo chown postgres:postgres /opt/obrc_ramfs/psql
```
