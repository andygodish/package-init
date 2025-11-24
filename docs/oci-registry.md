# OCI Registry Image Layer Storage

Your local Docker runtime is going to store image layers in a different format than an OCI registry.

## Docker local storage = uncompressed layers (diff IDs)

Docker stores each filesystem change (“layer”) as an uncompressed filesystem diff. These are represented by diff IDs:

```bash
sha256:<uncompressed-diff-hash>
```

These can be viewed by running `docker image inspect <image>` and looking for the `RootFS.Layers[]` field:

```bash
docker image inspect registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0 | jq '.[0] | {Id, RootFS}'
{
  "Id": "sha256:b00da6c6c771134fbcc8b83a9299c3f719ce36f325575b2d0cf82ef6dfb979b9",
  "RootFS": {
    "Type": "layers",
    "Layers": [
      "sha256:4470c8636ef8d59ecd85925ad81ff603b150c7b82e82b0e5d5ff653ec51e0d36",
      "sha256:4249584a096df8703bdda16157bfa7e176d1c774d6399db22900e33a3b1c1efe",
      "sha256:2b65a1778ad00f1508ee015b9f39cacffc6d2445fcc07aa7774f91476642ed9d"
    ]
  }
}
```

These 3 layers correspond to the 3 layers that can be seen in the output of an image's `docker history`:

```bash
docker history registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0                                 
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
b00da6c6c771   10 days ago   /bin/sh -c #(nop) LABEL "maintainer"="ironba…   1.32MB    
<missing>      10 days ago   /bin/sh -c #(nop) ENTRYPOINT ["busybox"]|cla…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) USER appuser|clamp-mtime=1…   0B        
<missing>      10 days ago   |11 BUNDLE_USER_CONFIG=/etc/.bundle/config C…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) COPY file:acf4fe8a5bf3345e…   0B        
<missing>      10 days ago   |11 BUNDLE_USER_CONFIG=/etc/.bundle/config C…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) USER root|clamp-mtime=1763…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) WORKDIR /app|clamp-mtime=1…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG BUNDLE_USER_CONFIG COM…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG BUNDLE_USER_CONFIG COM…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GEMRC GO…   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      10 days ago   /bin/sh -c #(nop) ARG COMPOSER_HOME HTTP_PRO…   0B        FROM registry1.dso.mil/ironbank/redhat/ubi/ubi9-minimal:9.7
<missing>      7 days ago    /bin/sh -c #(nop) LABEL "maintainer"="ironba…   67.3MB    
<missing>      7 days ago    /bin/sh -c #(nop) CMD ["/bin/bash"]|clamp-mt…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ENV PATH /usr/local/sbin:/…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ENV container oci|clamp-mt…   0B        
<missing>      7 days ago    |11 BUNDLE_USER_CONFIG=/etc/.bundle/config C…   0B        
<missing>      7 days ago    |11 BUNDLE_USER_CONFIG=/etc/.bundle/config C…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) COPY multi:a32f0e456b63435…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) COPY file:47b0625ed5b30ad5…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) COPY dir:076f7c662474d4353…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG BUNDLE_USER_CONFIG COM…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG BUNDLE_USER_CONFIG COM…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GEMRC GO…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME GOPROXY …   0B        
<missing>      7 days ago    /bin/sh -c #(nop) ARG COMPOSER_HOME HTTP_PRO…   0B        FROM registry.access.redhat.com/ubi9-minimal:9.7
<missing>      7 days ago    /bin/sh -c #(nop) LABEL "architecture"="x86_…   111MB     
<missing>      7 days ago    /bin/sh -c #(nop) COPY file:fde1a325755d265b…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) COPY file:fde1a325755d265b…   0B        
<missing>      7 days ago    /bin/sh -c #(nop) COPY file:93583a9ebbaeff1e…   0B 
```

Docker history shows a ton of “layers” (many 0B) while docker image inspect → `.RootFS.Layers` only shows 3 digests in this example image?

Docker history is a log of build instructions, not a list of actual filesystem layers whereas RootFS.Layers is a list of real filesystem layers. These are related, but absolutely not 1:1.

The docker history output is a chronological list of Dockerfile instructions from, but not all of those instructions actually create a new layer in the filesystem (ex: ENV, ARG, CMD, ENTRYPOINT, USER, WORKDIR, LABEL metadata). The 3 layers in RootFS.Layers correspond to the 3 instructions that actually created filesystem changes (COPY instructions in this case). They are actual `filesystem layers` in this final image.

When you push an image to an OCI registry, each layer is:

1. Tarred up
2. Compressed (usually with gzip)
3. Re-hashed --> this becomes the content digest

Diff IDs ≠ registry blob digests. The compression also explains why there is a discrepancy between the blob size in the OCI registry backend (Minio/S3) and the size shown in `docker history` or `docker image inspect` (uncompressed size).

## OCI registry = compressed layers (content digests)

You can see the compressed layer digests (content digests) by pulling the image manifest from its registry. The 3 layers you see in this example are the compressed versions (digests) of the 3 uncompressed layers (diff IDs) seen earlier:

```bash
docker pull registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0

v1.37.0: Pulling from ironbank/frontiertechnology/cortex/busybox
d80f4d75a127: Already exists 
7d6ca59745ac: Download complete 
bc59583073bb: Download complete 
Digest: sha256:b00da6c6c771134fbcc8b83a9299c3f719ce36f325575b2d0cf82ef6dfb979b9
Status: Downloaded newer image for registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0
registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0
```

These correspond to the following content digests seen in the image manifest:

> NOTE: The manifest inspect command reaches out to the registry to fetch the image manifest, a connection to the registry is required.

```bash
docker manifest inspect --verbose registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0
{
        "Ref": "registry1.dso.mil/ironbank/frontiertechnology/cortex/busybox:v1.37.0",
        "Descriptor": {
                "digest": "sha256:b00da6c6c771134fbcc8b83a9299c3f719ce36f325575b2d0cf82ef6dfb979b9",
        "OCIManifest": {
                ...
                },
                "layers": [
                        {"digest": "sha256:7d6ca59745ac48971cbc2d72b53fe413144fa5c0c21f2ef1d7aaf1291851e501"},
                        {"digest": "sha256:bc59583073bb8f33ed21c4bec47aa04a20555e09cb3da9e037f5317b7f6d3c86"},
                        {"digest": "sha256:d80f4d75a127e4a9effe0134e5264dad46514f8a6d88aa5af4e1fd64e386654c"},
                ],
                ...
}
```

In your OCI registry backend (Minio/S3), you will see these compressed blobs stored under their content digests.

"digest": "sha256:7d6ca59745ac48971cbc2d72b53fe413144fa5c0c21f2ef1d7aaf1291851e501 maps to zarf-registry/docker/registry/v2/blobs/sha256/7d/7d6ca59745ac48971cbc2d72b53fe413144fa5c0c21f2ef1d7aaf1291851e501/data in your Minio/S3 bucket.
