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
  ssh -i $PRIVATE_KEY root@minio-$i.$DOMAIN "ROOT_PASSWORD=$ROOT_PASSWORD DOMAIN=$DOMAIN SERVER_URL=$SERVER_URL CERTBOT_EMAIL=$CERTBOT_EMAIL time ./bootstrap && reboot"
done
```

### Installation notes

MinIO will run as the `minio` service added to systemd via `/lib/systemd/system/minio.service`.

The configuration can be found at `/etc/default/minio`.

Logs can be tailed via `journalctl -fu minio`.

## Ensure MinIO is running

After all nodes have been bootstrapped and rebooted. MinIO should be up and running. Let's verify that by checking the status of the MinIO service:

```bash
for i in {1..4}; do
  ssh -i $PRIVATE_KEY root@minio-$i.$DOMAIN "
    systemctl status minio --lines 0 &&
    mc alias set local https://minio-$i.$DOMAIN root $ROOT_PASSWORD
    mc admin info local
  "
done
```

## Add suppport for bucket encryption

MinIO supports SSE bucket encryption for full encryption at rest compliance. This requires adding and configuring [KES](https://github.com/minio/kes) on our cluster. To do this up we will run the `add-kes` script on each node.

Note that this script assumes using AWS Secrets Manager to store the main encryption keys. See documentation at https://min.io/docs/minio/container/operations/server-side-encryption/configure-minio-kes-aws.html for details.

Feel free to integrate using any other support secret manager by modifying the script.

The default key name used to encrypt all objects will be `minio-backend-default-key`. You can override the key on a per bucket basis if desired. Or tweak the script if you want a different name or disable cluster-wide default encryption.

When preparing the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`, ensure that the IAM user has the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "minioSecretsManagerAccess",
      "Action": [
        "secretsmanager:CreateSecret",
        "secretsmanager:DeleteSecret",
        "secretsmanager:GetSecretValue",
        "secretsmanager:ListSecrets"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "minioKmsAccess",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:Encrypt"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

Run the add-kes script on all nodes:

```bash
AWS_REGION=<aws-region>
AWS_ACCESS_KEY_ID=<aws-access-key-id>
AWS_SECRET_ACCESS_KEY=<aws-secret-access-key>

for i in {1..4}; do
  scp -i $PRIVATE_KEY bin/add-kes root@minio-$i.$DOMAIN:
  ssh -i $PRIVATE_KEY root@minio-$i.$DOMAIN "AWS_REGION=$AWS_REGION AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY ./add-kes"
done
```

Now we will have to separately restart minio on all the nodes to apply the changes. The restart was not included in the script since the restart command will freeze the script until a majority of nodes have been found to include the new configuration settings.

Note the ampersand at the end of the command to run it on multiple servers in the background.

```bash
for i in {1..4}; do
  ssh -i $PRIVATE_KEY root@minio-$i.$DOMAIN "systemctl restart minio" &
done
```

## Benchmarking MinIO

To benchmark MinIO we can use the S3 benchmarking tool [warp](https://github.com/minio/warp).

> [!NOTE]
> It's highly likely you'll end up bottlenecked by the network speed of the server you are running the benchmark from. Eg. if you're running on a 10Gbit network you should see around 1.25GB/s cluster performance. But the benchmark can still serve as a good indication if your cluster is healthy and all disks are functioning as expected. If you want to push the limits of your cluster, you can try the more complicated [distributed benchmark](https://github.com/minio/warp?tab=readme-ov-file#distributed-benchmarking) feature.

SSH into one of the MinIO nodes or an adjecent server on a fast network and run the following commands:

```bash
wget https://github.com/minio/warp/releases/download/v0.8.0/warp_Linux_x86_64.deb
dpkg -i warp_Linux_x86_64.deb

warp mixed \
  --host=minio-{1..4}.<domain> \
  --access-key=<minio-access-key> \
  --secret-key=<minio-access-secret> \
  --tls
```
