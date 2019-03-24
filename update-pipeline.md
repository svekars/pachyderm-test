# Modify your Pachyderm pipeline

This tutorial walks you through the steps required to update your
Pachyderm pipeline.

* This tutorial assumes that you have installed Pachyderm as described in
[Local Intallation](https://pachyderm.readthedocs.io/en/latest/getting_started/local_installation.html)
and completed the [Beginner Tutorial](https://pachyderm.readthedocs.io/en/latest/getting_started/beginner_tutorial.html).
* You need to complete all sections on this page to successfully make
changes in your pipeline.
* In specific operating systems, you might need to run some of the commands
below as root by using `sudo`.

## Image processing

Pachyderm uses the Open Source Computer Vision Library (OpenCV)
and [Matplotlib](https://matplotlib.org) library for edge detection
and image processing. Specifically, OpenCV uses the Canny edge
detection algorithm for this purpose.

The code for the edge detection pipeline is packaged as a
Python script called `edges.py`:

```bash
# edges.json
{
  "pipeline": {
    "name": "edges"
  },
  "transform": {
    "cmd": [ "python3", "/edges.py" ],
    "image": "pachyderm/opencv"
  },
  "input": {
    "pfs": {
      "repo": "images",
      "glob": "/*"
    }
  }
}
```

## Modify a pipeline

Apply a small change to the `edges.py` pipeline, such as change
the color of the background and the contour line. OpenCV uses the
`ColorMap` parameter to define a color scheme for the edge definition
pipeline. In the Python script below, this parameter is specified as
`cmap`.

```bash
import cv2
import numpy as np
from matplotlib import pyplot as plt
import os
 
# make_edges reads an image from /pfs/images and outputs the result of running
# edge detection on that image to /pfs/out. Note that /pfs/images and
# /pfs/out are special directories that Pachyderm injects into the container.
def make_edges(image):
    img = cv2.imread(image)
    tail = os.path.split(image)[1]
    edges = cv2.Canny(img,100,200)
    plt.imsave(os.path.join("/pfs/out", os.path.splitext(tail)[0]+'.png'), edges, cmap = 'gray')

# walk /pfs/images and call make_edges on every file found
for dirpath, dirs, files in os.walk("/pfs/images"):
    for file in files:
        make_edges(os.path.join(dirpath, file))
```

To verify that the pipeline is modified, complete the following steps:

1. Change the `cmap` parameter. For example:

  ```bash
  cmap = 'jet'
  ```

1. Save your changes.

For more information about color maps available in OpenCV, see
[OpenCV documentation](https://docs.opencv.org/3.4/d3/d50/group__imgproc__colormap.html)

## Build a new Docker image

After you apply your changes, you need to build a new Docker
image by using the Dockerfile in the `opencv` directory and then push
it to your Docker registry.

OpenCV requires sufficient memory, CPU, and swap space available in
your Docker virtual machine (VM). Before you start building a new image
assign, verify that your Docker VM resources match or exceed the following
minimal requirements:

* CPU x 4
* Memory x 12 GB
* Swap x 2 Gb

If you get the following error, your Docker VM does not have enough resources
to build the image:

```bash
c++: internal compiler error: Killed (program cc1plus)
Please submit a full bug report,
with preprocessed source if appropriate.
See <file:///usr/share/doc/gcc-7/README.Bugs> for instructions.

```

To build a new Docker image, complete the following steps:

1. Change the directory to the directory where your Dockerfile is located.

1. Build a new image from the Dockerfile.

   ```bash

   docker build - < Dockerfile
   ```

   This operation might take some time.

## Log in to Docker registry

Before you can upload your image to a Docker registry, you need to create
one, create a repository there, and then log in to it from your command
line interface. If you already have a registry, skip the steps that describe
how to create a Docker registry and proceed to the login instructions. 

In this example, we use a public repository on DockerHub, but you can use any
registry of your choice.

**Procedure:**

* If you do not have a Docker registry.

  1. Go to https://hub.docker.com.
  1. Create an account.
  1. After you log in, create a public repository by clicking
  **+Create Repository** and following the on-screen instructions.

1. Log in to DockerHub on your machine:

   ```bash
   docker login --username=<dockerhub-username> --password=<dockerhub-password> <dockerhub-fqdn>
   ```

   **Example:**

   ```bash
   docker login --username=test --password=containersareawesome123 https://index.docker.io/v1/
   ```

## Upload your image to a Docker image registry

After you build your image, you need to tag it and upload it
to your Docker registry. In this example, we use a public repository on
DockerHub.

**Procedure:**

1. View the list of Docker images:

   ```bash
   docker image ls
   ```

   **Example of system response:**

   ```bash
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   none                none                dc97ecd2ad5f        6 minutes ago       896MB
   ubuntu              18.04               94e814e2efa8        12 days ago         88.9MB
   ```

   The latest created image appears with a `none` tag.

1. Tag your image:

   ```bash
   docker tag <IMAGE_ID>:<TAG> <REGISTRY>>
   ```

   **Example:**

   ```bash
   docker tag dc97ecd2ad5f test/opencv:latest
   ```

1. Verify that your image was successfully tagged:

   ```bash
   docker image ls
   ```

   **Example of system response:**

   ```bahs
   REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
   test/opencv         latest              dc97ecd2ad5f        6 minutes ago       896MB
   ubuntu              18.04               94e814e2efa8        12 days ago         88.9MB

1. Upload your image to your registry:

   ```bash
   docker push <image>:tag
   ```

1. Go to your registry repository to verify that your image was
successfully uploaded.

## Update your pipeline

After you upload your image to your registry, you need to rerun the
`edges.py` pipeline to see your changes.

**Procedure:**

1. Point the `edges.json` file to the new image and registry:

   ```bash
   # edges.json
   {
     "pipeline": {
       "name": "edges"
     },
     "transform": {
       "cmd": [ "python3", "/edges.py" ],
       "image": "test/opencv"
     },
     "input": {
       "pfs": {
         "repo": "images",
         "glob": "/*"
       }
     }
   }

   In the example above the `image` parameter was updated to
   `"image": "test/opencv"`.

1. Rerun the `edges.py` pipeline:

   ```bash
   pachctl update-pipeline -f edges.json
   ```
1. Verify that `pachctl` created a new job for the pipeline:

   ```bash
   pachctl list-job
   ```

   **Example of system response:**

   ```bash
   ID                               PIPELINE STARTED           DURATION           RESTART PROGRESS  DL UL STATE
   672ea806a8d94327ad498c0fb8695bf7 edges    7 seconds ago     -                  0       0 + 0 / 1 0B 0B running
   ```

1. After the pipeline completes, view your image:

   * On macOS, run:

     ```bash
     pachctl get-file edges master liberty.png | open -f -a /Applications/Preview.app
     ```

   * On Linux, run:

     ```bash
     pachctl get-file edges master liberty.png | display
     ```
   The processed image must reflect the changes you applied to the pipeline.

## Troubleshooting

* `pachctl` does not create a job for the pipeline.

  To troubleshoot, complete the following steps:

  1. Verify that the pipeline container is up and running:

     ```bash
     kubectl get all
     ```
     **Example of system response:**

     ```bash
     NAME                            READY   STATUS             RESTARTS   AGE
     pod/dash-655844658b-7g4jc       2/2     Running            2          10d
     pod/etcd-6b866844bd-jmzwj       1/1     Running            1          10d
     pod/pachd-774488bc98-x4647      1/1     Running            1          10d
     pod/pipeline-edges-v1-wp95s     1/2     CrashLoopBackOff   60         10d
     ```

     In the example output above the `edges` pipeline is in `CrashLoopBackOff`.
     Run `kubectl describe pod <pod-name>` to obtain more information about the
     issue.

* `failure: failed to process datum:...`

  When you see this error message, it means that the code changes in the pipeline
  might be incorrect and causing the pipeline to fail. Double-check your changes.
