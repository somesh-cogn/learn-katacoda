The goal of this lab is to get a basic understanding of the three Open Containers Initiative (OCI) specificaitons that govern finding, running, building and sharing container - image, runtime, and distribution. At the highest level, containers are two things - files and processes - at rest and running. First, we will take a look at what makes up a [Container Repository](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/#h.20722ydfjdj8) on disk, then we will look at what directives are defined to create a running [Container](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/#h.j2uq93kgxe0e).

If you are interested in a slightly deeper understanding, take a few minutes to look at the OCI  work, it's all publicly available in GitHub repositories:

- [The Image Specification Overview](//github.com/opencontainers/image-spec/blob/master/spec.md#overview)
- [The Runtime Specification Abstract](//github.com/opencontainers/runtime-spec/blob/master/spec.md)
- [The Distributions Specification Use Cases](https://github.com/opencontainers/distribution-spec/blob/master/spec.md#use-cases)

Now, lets run some experiments to better understand these specifications.

## The Image Specification

First, lets take a quick look at the contents of a container repository once it's uncompressed. We will use a utility you may have seen before called Podman. The syntax is nearly identical to Docker. Create a working directory for our experiment, then make sure the fedora image is cached locally:

``mkdir fedora
cd fedora
podman pull fedora``{{execute}}

Now, export the image to a tar, file and extract it:
``podman save -o fedora.tar fedora
tar xvf fedora.tar``{{execute}}

Finally, let's take a look at three important parts of the container repository - these are the three major pieces that can be found in a container repository when inspected:

1. Manifest - Metadata file which defines layers and config files to be used
2. Config - Config file which is consumed by the container engine. This config file is combined with engine defaults and user inputs (command line options to th engine) to create the runtime Config.json which is eventually handed to the continer runtime (runc)
3. Image Layers - tar files, typically gzipped which when merged together create a root file system which is mounted at container creation

In the Manifest, you should see one or more Config and Layers entries:

``cat manifest.json``{{execute}}

In the Config file, notice all of the meta data that looks strikingly similar to command line options in Docker & Podman:

``cat $(cat manifest.json | awk -F 'Config' '{print $2}' | awk -F '["]' '{print $3}')``{{execute}}

Each Image Layer is just a tar file. When all of the necessary tar files are extracted into a single directory, they can be mounted into a container's mount namespace:

``tar tvf $(cat manifest.json | awk -F 'Layers' '{print $2}' | awk -F '["]' '{print $3}')``{{execute}}

The take away from inspecting the three major parts of a container repository is that they are really just the wittiest use of tarballs ever invented. Now, that we understand what is on disk, lets move onto the runtime.

# The Runtime Specification

This specification governs the format of the file that is passed to container runtime. This is typically runc, but every OCI compliant runtime will accept this file format (Examples: Kata, gVisor, etc). Typically, this file is constructed by a container engine such as CRI-O, Podman, or the Docker engine. These files can be created manually, but it's a tedious process. Instead, we are going to do couple of experiments so that you can get a feel for this file without having to create one manually. 

The runc tool, which is the OCI reference implementation of a [Container Runtime](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction/#h.6yt1ex5wfo55), has the ability to create a very simple spec file. Create one and take a quick look at the fairly simple set of directives:

``runc spec
cat config.json``{{execute}}

Now, lets steal a more complex file from podman. Create a long running container (aka like a daemon or service):

``podman run -dt fedora bash``{{execute}}

Now, lets steal the config.json which Podman created. Again, this file is a combination of inputs from the:

1. The container repository, the config.json which we inspected before. Think of this as a set of defaults that are created by the image builder. They are a combination of user inputs (Example: CMD) and defaults specified by the build tool (Example: Architecture)

2. The container engine itself. Some of these can be configured in the configuration for the container engine (Example: SECCOMP profiles), some are dynamically generated by the container engine (Example: sVirt/SELinux contexts, or Bind Mounts - aka the copy on write layer which gets mounted in the container's namespace), while others are hardcoded in the engine (Example: the default namespaces to utilize).

3. The command line options specified by the user of the container engine (or robot in Kubernetes' case). Some of these are simple things like Bind mounts (Example: -v /data:/data) or more complex like security options (Example: --privileged which disables a lot of technologies in the kernel). 

Take a look at this example config.json in all of its glory. See if you can spot directives which come from the container repository, the engine, and the user:

``cat $(find /var/lib/containers/ | grep  $(podman ps --no-trunc -q | tail -n 1)/userdata/config.json)``{{Execute}}

Now that we have a basic understanding, lets move on to starting a container...
