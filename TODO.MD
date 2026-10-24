1. RustFS
So I need to install this on my linux laptop. There is a docker option so I think i'm going to kill two birds with one stone and force myself into the docker route to get better with docker

So we need to pull the image
docker pull rustfs/rustfs:latest

Then we create a persistent storage so it stays available even if we replace the container

Then we replace the credential placeholders for the below and then run the container

docker run -d \
  --name rustfs \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -v rustfs-data:/data \
  -e RUSTFS_ACCESS_KEY="<your-access-key>" \
  -e RUSTFS_SECRET_KEY="<your-secret-key>" \
  -e RUSTFS_ADDRESS=":9000" \
  -e RUSTFS_CONSOLE_ADDRESS=":9001" \
  -e RUSTFS_CONSOLE_ENABLE=true \
  -e RUSTFS_OBS_LOGGER_LEVEL=error \
  -e RUSTFS_OBS_LOG_DIRECTORY="/var/log/rustfs/" \
  rustfs/rustfs:latest \
  /data

important notice from the docs
Set credentials when creating the container

Set unique RUSTFS_ACCESS_KEY and RUSTFS_SECRET_KEY environment variables before exposing RustFS to a network. Do not use the well-known rustfsadmin value for either credential. If the container was started without custom credentials, stop and recreate it with both -e options shown above; the rustfs-data volume remains intact.

Then we need to bind a host directory. The below is imoprtant to ensure permissions are correct

 If you mount a host directory instead (-v /path/on/host:/data), keep in mind that the container runs as non-root user rustfs with id 10001. Make the host directory owned by 10001, otherwise you will encounter permission denied errors:

 chown -R 10001:10001 /path/to/host_directory

We then need to verify the deployment:
docker ps --filter name=rustfs
curl --fail http://localhost:9000/health
The S3 API is available at http://localhost:9000, and the Console is available at http://localhost:9001

So there is already a docker compose file according the docs
The RustFS repository ships a docker-compose.yml that includes grafana, prometheus, otel-collector, and jaeger services, mainly for observability. To deploy RustFS together with these services, clone the RustFS code repository locally:
So we can clone this and then paste it into our project

git clone https://github.com/rustfs/rustfs.git

Then we bring up the docker container

docker compose --profile observability up -d

The compose file uses an initialization container to grant the correct access rights to rustfs: the rustfs_perms service below changes the ownership of the mounted volumes to 10001 before rustfs starts, using depends_on to wait for it to complete. To keep logs persistent and accessible, the host log directory is mapped to the container's /var/log/rustfs/ path:

As part of this project I will be writing the logs elsewhere so I won't be using the below log option but I'm keeping a record for now to modify later on

  services:
    # grant the necessary permissions to RUSTFS volumes path
    rustfs_perms:
      image: alpine
      user: root
      volumes:
        - /path/to/host_directory/volumes:/fix_path
      command: chown -R 10001:10001 /fix_path

    rustfs:
      image: rustfs/rustfs:latest
      depends_on:
        rustfs_perms:
          condition: service_completed_successfully
      volumes:
        - /path/to_host_directory/volumes/data:/data
        - /path/to_host_directory/volumes/logs:/var/log/rustfs/
      environment:
        - RUSTFS_ADDRESS=":9000"
        - RUSTFS_CONSOLE_ADDRESS=":9001"
        - RUSTFS_CONSOLE_ENABLE=true
        - RUSTFS_OBS_LOGGER_LEVEL=error
        - RUSTFS_OBS_LOG_DIRECTORY="/var/log/rustfs/"

      # ... other configurations

The s3 api srver is served here - http://localhost:9000

If I needed multi node deployment I can use the below but I think for this project it will just be 1. I'm saving this as reference anyhow

docker run -d \
  --name rustfs \
  --network host \
  -v /mnt/rustfs/data:/data \
  -e RUSTFS_ACCESS_KEY="<your-access-key>" \
  -e RUSTFS_SECRET_KEY="<your-secret-key>" \
  -e RUSTFS_ADDRESS=":9000" \
  -e RUSTFS_CONSOLE_ADDRESS=":9001" \
  -e RUSTFS_CONSOLE_ENABLE=true \
  -e RUSTFS_OBS_LOGGER_LEVEL=error \
  -e RUSTFS_OBS_LOG_DIRECTORY="/var/log/rustfs/" \
  -e RUSTFS_VOLUMES="http://node{1...4}:9000/data/rustfs{0...3}" \
  rustfs/rustfs:latest

But to do this you have to add the notes to the hostfile

192.168.1.1 node1
192.168.1.2 node2
192.168.1.3 node3
192.168.1.4 node4

if using TLS - I'm not actually sure if I will be so I'll query this
-v /path/to/certs:/certs \
-e RUSTFS_TLS_PATH=/certs \

Once we're configured we open the deployment in a web browser: These are the login methods

Key Login uses the access key and secret key configured for the RustFS deployment. This is the standard login method for a local administrator.
STS Login uses temporary Security Token Service (STS) credentials. Use it only when your identity workflow has issued a valid session token.
OIDC Login appears when an OpenID Connect (OIDC) provider is configured. Select the provider and complete authentication with the identity provider.

I'll be configuring private secrets/keys via a .env file or might even see if there's a better option

At this point I think I'll create a bucket and then set up a script to deploy something and read something just to get a feel for it as I've never used this before but it looks easy enough. There is an "RC" option that looks interesting so I might challenge myself to see if I can do it this way instead of the gui. There is an API but I'll be leaving any API stuff to the end if at all

rc bucket create rustfs/my-bucket
rc bucket list rustfs/

S3 requests must be signed with AWS Signature V4, so use an S3 client rather than hand-crafting headers. With the AWS CLI configured for your access keys:
Copy Text

aws s3api create-bucket \
  --bucket bucket-creation-by-api \
  --endpoint-url http://localhost:9000

This is how we set a quota:
rc bucket quota set rustfs/my-bucket 1G

I'll be utilising the results so I need the data in a json file
rc bucket quota info rustfs/my-bucket --json

if I need to adjust the quota I can clear it with the below

rc bucket quota clear rustfs/my-bucket
rc bucket quota info rustfs/my-bucket

rc 0.1.29 does not expose commands for detailed usage statistics or advisory checks for proposed writes. Use the RustFS Admin API for these operations. Requests must use AWS Signature Version 4 with an active RustFS credential.

The examples use these shell variables:
Copy Text

export RUSTFS_ENDPOINT=http://localhost:9000
export RUSTFS_ACCESS_KEY=<your-access-key>
export RUSTFS_SECRET_KEY=<your-secret-key>
export BUCKET_NAME=my-bucket

Read detailed usage statistics
Copy Anchor Link

Use the statistics endpoint to retrieve the limit, current usage, remaining bytes, and percentage used:
Copy Text

curl --fail-with-body \
	--aws-sigv4 "aws:amz:us-east-1:s3" \
	--user "${RUSTFS_ACCESS_KEY}:${RUSTFS_SECRET_KEY}" \
	"${RUSTFS_ENDPOINT}/rustfs/admin/v3/quota-stats/${BUCKET_NAME}"

the rest of the bucket docs are here but the above covers all I should need:
https://docs.rustfs.com/en/administration/data/bucket/creation
