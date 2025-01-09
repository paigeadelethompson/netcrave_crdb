# Quickstart 
Create a directory to contain data and certificate files 
- `mkdir crdb`
- `cd crdb`

## Create CA
```
docker run -it --rm -v $(pwd)/crdb/certs:/certs -v $(pwd)/crdb/my-safe-directory:/my-safe-directory \
docker.io/cockroachdb/cockroach:latest cert create-ca --certs-dir=/certs --ca-key=/my-safe-directory/ca.key
```

- `tar -cvf - certs/ | gzip -9 > ca.tar.GZ` and copy this to the `crdb` directory on each node that will participate in the cluster.

## Create server certificates

Run the following for every host that will be part of the cluster: 
```
docker run -it --rm -v $(pwd)/crdb/certs:/certs -v $(pwd)/crdb/my-safe-directory:/my-safe-directory \
docker.io/cockroachdb/cockroach:latest cert create-node srv-1-1.netcrave.io \
srv-1-1.tailedb13e.ts.net 100.97.79.124 fd7a:115c:a1e0::a801:4f7d 1.1.1.1 \
2000:1:2:3:4:: 127.0.0.1 localhost srv-1-1 \
--certs-dir=/certs --ca-key=/my-safe-directory/ca.key
```

and replace: 

- `1.1.1.1` and `2000:1:2:3:4::` with your public IPv4 and IPv6 addresses (or omit them if you only wish to use private addresses
- `srv-1-1` and `srv-1-1.netcrave.io` with the correct hostnames for the cluster node
- `srv-1-1.tailedb13e.ts.net`, `100.97.79.124`, and `fd7a:115c:a1e0::a801:4f7d` are specific to Tailscale and should be changed to match
your own or omitted if you don't intend to use Tailscale.

## Start each node in the cluster 

```
 docker run -it -d --restart always --net host -v $(pwd)/crdb/data:/cockroach/cockroach-data \
-v $(pwd)/crdb/certs:/certs docker.io/cockroachdb/cockroach:latest start --certs-dir=/certs \
--join=srv-1-2.netcrave.io,srv-1-3.netcrave.io --listen-addr=:26257 \
--http-addr=100.97.79.124:443 --advertise-addr=srv-1-1.netcrave.io
```

- `--http-addr=100.97.79.124:443` can be changed to `--http-addr=:443` to listen on all addresses if necesarry, it's the administrative web service

These options vary for each node in the cluster:
- `--advertise-addr=srv-1-1.netcrave.io` should be a reachable FQDN of the current node that Cockroach is being started on
- `--join=srv-1-2.netcrave.io,srv-1-3.netcrave.io` should list all other nodes participating in the cluster

## Initialize the cluster
After all nodes have been started, run the following on the first node: 

```
docker run -it --rm --net host -v $(pwd)/crdb/certs:/certs docker.io/cockroachdb/cockroach:latest \
init --certs-dir=/certs
```

This step only needs to be performed once 

## Create the root client certificate 

```
docker run -it --rm -v $(pwd)/crdb/certs:/certs -v $(pwd)/crdb/my-safe-directory:/my-safe-directory \
docker.io/cockroachdb/cockroach:latest cert create-client root --certs-dir=/certs --ca-key=/my-safe-directory/ca.key
```

### Install enterprise license
Start a SQL shell: 
```
docker run -it --rm --net host -v $(pwd)/dump/:/dump -v $(pwd)/crdb/data:/cockroach/cockroach-data \
-v $(pwd)/crdb/certs:/certs docker.io/cockroachdb/cockroach:latest sql --certs-dir /certs
```

Run the following SQL commands:

```
SET CLUSTER SETTING cluster.organization = 'Netcrave Communications';
SET CLUSTER SETTING enterprise.license = '<secret>';
```

Replace `<secret>` with your license key provided by Cockroach Labs. 

### Create a user 

Create a certificate for the user: 
```
docker run -it --rm -v $(pwd)/crdb/certs:/certs -v $(pwd)/crdb/my-safe-directory:/my-safe-directory \
docker.io/cockroachdb/cockroach:latest cert create-client my_test_user --certs-dir=/certs --ca-key=/my-safe-directory/ca.key
```

Start a SQL Shell, then run the following SQL commands: 
```
CREATE USER my_test_user;
```

To create a user with a password: 
```
CREATE USER my_test_user WITH LOGIN PASSWORD '$tr0nGpassW0rD'
```

### Create a database and grant privileges to user

Start a SQL Shell, then run the following SQL commands: 
```
CREATE DATABASE my_test_db;
GRANT ALL ON DATABASE my_test_db TO my_test_user WITH GRANT OPTION;
```

### Set default transaction isolation level for a database 
This step requires an enterprise license: 

In the SQL shell:
```
ALTER DATABASE my_test_db SET default_transaction_isolation = 'read committed';
```

## Database connection string

- `postgresql://my_test_user@srv-1-1.netcrave.io:26257/my_test_db?sslcert=/crdb/client.my_test_user.crt&sslkey=/crdb/client.my_test_user.key&sslmode=verify-full&sslrootcert=/crdb/ca.crt`
- With password: `postgresql://my_test_user:$tr0nGpassW0rD@srv-1-1.netcrave.io:26257/my_test_db?sslcert=/crdb/client.my_test_user.crt&sslkey=/crdb/client.my_test_user.key&sslmode=verify-full&sslrootcert=/crdb/ca.crt`

# TODO
- Setup multi-tenancy (with enterprise encryption)
- Follow PL/PGSQL support status https://github.com/cockroachdb/cockroach/issues/137561
