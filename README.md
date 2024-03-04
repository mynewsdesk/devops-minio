# devops-minio

This repository contains documentation, tools and scripts for managing our self hosted S3 object storage service via MinIO on Hetzner.

## Install Ubuntu

Start by installing Ubuntu 22.04 via the Hetzner rescue OS. Note that `HOSTNAME` needs to be unique and use the prefix minio- followed by the number of the node. The following `installimage` config assumes using SX134 server with 2x NVMe drives in to host the OS.

```bash
HOSTNAME="minio-<1..n>"

cat <<EOF > installimage-minio
DRIVE1 /dev/nvme0n1
DRIVE2 /dev/nvme1n1
SWRAID 1
SWRAIDLEVEL 1
HOSTNAME $HOSTNAME
USE_KERNEL_MODE_SETTING yes
PART swap swap 4G
PART /boot ext3 1024M
PART / ext4 all
IMAGE /root/.oldroot/nfs/install/../images/Ubuntu-2204-jammy-amd64-base.tar.gz
EOF

installimage -a -c installimage-minio && reboot
```

## Bootstrap MinIO

Start by adding DNS A records for each MinIO node. The expected format is `minio-<1..n>.<your-domain>`.

If you're using Cloudflare, don't enable the proxy since we expect the servers to manage their own TLS certificates.

Prepare ENV variables required by the bootstrap script:

```bash
# Clean alphanumeric password, must be the same for all nodes
ROOT_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | cut -c1-32)

# The domain name used for the MinIO nodes
DOMAIN=<your-domain>

# SERVER_URL is the public URL of the MinIO server and must be the same across all nodes.
# Either use a load balancer endpoint or use a single minio node directly.
SERVER_URL=https://minio-1.$DOMAIN

# Email used for Let's Encrypt certificate generation
CERTBOT_EMAIL=devops@$DOMAIN
```

Make sure that the `hostname` within each of your Ubuntu installation is set to the `minio-` prefix followed by node number, ie: `minio-<1..n>`. If you followed the "Install Ubuntu" step correctly, this should be the case.

Run the bootstrap script on each MinIO node:

```bash
PRIVATE_KEY=~/.ssh/<minio-node-private-key>

for i in {1..4}; do
  scp -i $PRIVATE_KEY bin/bootstrap root@minio-$i.$DOMAIN:
  ssh -i $PRIVATE_KEY root@minio-$i.$DOMAIN "chmod +x ./bootstrap && ROOT_PASSWORD=$ROOT_PASSWORD DOMAIN=$DOMAIN SERVER_URL=$SERVER_URL CERTBOT_EMAIL=$CERTBOT_EMAIL time ./bootstrap && reboot"
done
```
