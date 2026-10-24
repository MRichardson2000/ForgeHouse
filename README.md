Welcome to ForgeHouse!

This is my Data Lakehouse Project.

This is going to push me out of my comfort zone because I have never worked in a data industry and therefore haven't seen the true intracacies involved with the kind of work undertaken. I've felt stuck on even getting started on this because I struggle to visualise what "good" looks like having not seen this sort of thing in an industry before. I'm throwing myself in the deep end, there will be a lot of learning to be done on this. My plan of action is in a spreadsheet in my google drive which im intentionally keeping out of this repo but this has the details for the process I will be following.

To get over the first hurdle I've put together a really basic folder structure and plan just so I can actually get some motion on this. For now I'll be placing the building blocks for this in order to get it moving along.


RustFS:

If I run the below it pauses the container:
docker stop forgehouse-storage

To turn it back on it run the below:
docker start forgehouse-storage

If I delete it with this it removes the container instance:
docker rm forgehouse-storage

This is when I would have to run the below:
docker run -d \
  --name forgehouse-storage \
  --restart unless-stopped \
  -p 9000:9000 \
  -p 9001:9001 \
  -v forgehouse-data:/data \
  -e RUSTFS_ACCESS_KEY="$RUSTFS_ROOT_USER" \
  -e RUSTFS_SECRET_KEY="$RUSTFS_ROOT_PASSWORD" \
  -e RUSTFS_ADDRESS=":9000" \
  -e RUSTFS_CONSOLE_ADDRESS=":9001" \
  -e RUSTFS_CONSOLE_ENABLE=true \
  -e RUSTFS_OBS_LOGGER_LEVEL=error \
  rustfs/rustfs:latest \
  /data

sudo docker ps will reveal the image

To check the logs:
sudo docker logs forgehouse-storage

When it's up, go here:
http://localhost:9001/

Login with the details in the mise file

We install RC via docker 
docker run --rm --network host rustfs/rc:latest --help

Make sure you have permissions before creating the volume:
docker run --rm \
  --user 0 \
  -v forgehouse-rc-config:/config \
  --entrypoint sh \
  rustfs/rc:latest \
  -c 'chown -R 100:101 /config && chmod -R u+rwX /config'

We create a persistent volume:
docker run --rm --network host \
  -v forgehouse-rc-config:/home/rc/.config/rc \
  -e RUSTFS_ACCESS_KEY="$RUSTFS_ROOT_USER" \
  -e RUSTFS_SECRET_KEY="$RUSTFS_ROOT_PASSWORD" \
  rustfs/rc:latest alias set local \
  http://localhost:9000 \
  "$RUSTFS_ROOT_USER" "$RUSTFS_ROOT_PASSWORD"

Verify the alias persists:
docker run --rm \
  -v forgehouse-rc-config:/home/rc/.config/rc \
  rustfs/rc:latest alias list

Create a bucket:
docker run --rm --network host \
  -v forgehouse-rc-config:/home/rc/.config/rc \
  rustfs/rc:latest bucket create local/test

Verify the bucket:
docker run --rm --network host \
  -v forgehouse-rc-config:/home/rc/.config/rc \
  rustfs/rc:latest bucket list local/
