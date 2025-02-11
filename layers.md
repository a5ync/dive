# Why?

When analyzing recently built images, **dive** crashes when trying to access compressed layers—a new feature introduced in Docker 20.10. This happens because **dive** relies on `dockerd` format to access image layers, but the image is stored in `containerd`, which **dive** does not fully support yet.

Extracting the image into a tarball and then loading it into Dive behaves the same way.

Docker Desktop now uses containerd by default instead of dockerd. You can check this in `Settings -> General -> Use containerd for pulling and storing images`.

When using the Homebrew version of **dive**, attempting to analyze an image fails:

```sh
/opt/homebrew/bin/dive 821bc09fd83f
Using config file: dive/.bouncer.yaml
Image Source: docker://821bc09fd83f
Fetching image... (this can take a while for large images)
cannot fetch image
could not find 'blobs/sha256/0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a' in parsed layers
```

Whereas a properly parsed image should look like this:

```sh
dive 821bc09fd83f
Using config file: dive/.bouncer.yaml
Image Source: docker://821bc09fd83f
Fetching image... (this can take a while for large images)
Analyzing image...
Building cache...
```

When extracting the image into a tar file and inspecting the layers, I can see that the problematic layer exists within the image, even though it is very small (93B in size).

```sh
docker save -o myimage.tar 821bc09fd83f
mkdir extracted_image
tar -xf myimage.tar -C extracted_image
ls -lah extracted_image/blobs/sha256
# -r--r--r--   1 paugustyn  staff    93B Dec 31  1969 0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a
```

Further inspection reveals that this particular layer is a **gzip (tar.gz) archive** containing just an empty directory:

```sh
file 0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a
0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a: gzip compressed data, original size modulo 2^32 1536
tar -tzf 0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a
app/
```

This corresponds to a layer created by the following **Dockerfile** step:

```dockerfile
WORKDIR /app
```


# Building Dive Locally for Testing

```sh
# download all dependencies
make bootstrap
make build
# confirmed the issue with the build
./snapshot/dive_darwin_arm64/dive 821bc09fd83f
# Image Source: docker://821bc09fd83f
# Fetching image... (this can take a while for large images)
# cannot fetch image
# could not find 'blobs/sha256/0366bbd1035ada4fba244e4de56b7e1988102531fee81e22dc822e143ce6c57a' in parsed layers
```

after removing the cap condition it works :)
