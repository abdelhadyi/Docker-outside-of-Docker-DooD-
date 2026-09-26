# Docker-outside-of-Docker (DooD) with Jenkins

If you've ever needed a containerized application to build or run other Docker containers — a CI server, an automation agent, a deployment tool, you've probably run into a strange requirement: **a container that needs to talk to Docker itself.**

There are two well-known patterns for this: **Docker-in-Docker (DinD)** and **Docker-outside-of-Docker (DooD)**. This repo focuses on DooD: what it is, why it's usually the better choice, and how to set it up — using a Jenkins container as a concrete example.

## The Problem

Say you're running Jenkins inside a container. A pipeline step needs to run `docker build` to create an image. But Jenkins is *inside* a container, it doesn't have Docker installed, and even if it did, running a full Docker daemon nested inside another container (DinD) brings real complications: it usually requires `--privileged` mode, has known volume and networking quirks, and duplicates image layers and caches that already exist on the host.

DooD solves this differently: instead of running a *second* Docker daemon inside the container, you let the container talk to the host's existing Docker daemon.

## The Core Idea

Docker's client-server architecture makes this possible. The `docker` CLI you type commands into is just a client, it doesn't do any of the actual container work itself. It sends instructions over a socket (usually `/var/run/docker.sock`) to the Docker daemon, which does the real work.

DooD works by:

1. Installing the Docker CLI only inside your container.
2. Mounting the host's Docker socket into the container.

Now, when your container runs a `docker` command, it's really just relaying that command to the daemon running on the host machine, as if you'd typed it there yourself. Any container it "creates" is actually created as a sibling on the host, not nested inside the original container.

## Step-by-Step: Setting Up DooD (Jenkins Example)

### Step 1: Confirm the host's Docker socket path

On most Linux hosts, this is:

```bash
/var/run/docker.sock
```

### Step 2: Install the Docker CLI inside your container image

For a Jenkins image, this typically means extending the base image with the Docker CLI package. In a Dockerfile:

```dockerfile
FROM jenkins/jenkins:lts

USER root
RUN apt-get update && \
    apt-get install -y docker.io && \
    usermod -aG docker jenkins

USER jenkins
```

Only the CLI binary is needed here — not a Docker daemon.

### Step 3: Mount the host's Docker socket when running the container

```bash
docker run -d \
  --name jenkins \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  -p 8080:8080 \
  my-jenkins-dood-image
```

The `-v /var/run/docker.sock:/var/run/docker.sock` line is the whole trick — it exposes the host's socket inside the container at the same path.

### Step 4: Handle permissions

The Docker socket is owned by a group (usually `docker`) on the host. The user running processes inside your container needs access to that same group ID or you'll hit "permission denied" errors. Two common fixes:

* Add the in-container user to a `docker` group with a matching GID to the host's.
* Run the container with `--group-add` pointing to the host's docker group GID.

### Step 5: Test it

From inside the running container:

```bash
docker ps
```

You should see:
  docker ps listing your jenkins container (because the CLI is using the host’s daemon)

### Step 6: Use it in your workflow

In Jenkins, a pipeline stage can now run Docker commands directly:

```groovy
stage('Build Image') {
    steps {
        sh 'docker build -t myapp:${BUILD_NUMBER} .'
    }
}
```

That `docker build` runs on the host daemon, producing an image that lives alongside Jenkins itself, not trapped inside Jenkins' own container filesystem.

## Key Things to Know

* **Containers built this way are siblings, not children.** Anything spawned via DooD lives at the same level as your original container on the host, not nested inside it. Volume mounts using relative container paths can be misleading because of this — always use host-absolute paths.
* **Security trade-off.** Mounting the Docker socket effectively gives the container root-equivalent control over the host. Anyone who can execute commands inside that container can, in practice, control the host's Docker daemon. This is a real concern in multi-tenant or untrusted environments.
