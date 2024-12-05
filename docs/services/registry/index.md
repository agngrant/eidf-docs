# Overview

The EIDF Container Image Registry is for EIDF Users to store their container images and use them on EIDF and other services.

The registry uses the [Harbor](https://goharbor.io/) for its platform. Further Harbor specific documentation can be found [here](https://goharbor.io/docs/2.11.0). 

Current Version: 2.11.0

## The Registry Interface and Accounts

All EIDF Users can access the registry through their SAFE account.

The registry can be logged into at [https://registry.eidf.ac.uk](https://registry.eidf.ac.uk). If you are not logged into SAFE, the registry will redirect you to SAFE.

User tokens can be access from the User Profile from the dropdown under your Username at the top right hand corner of the page. These will be needed to login from Docker or other container services.

## Projects

To create a project, use the New Project button.

Important - choose if your project will be private or public. Private projects will only be accessible to you and any individuals or groups you give access to. Public will be accessible by all users.

The project has a range of configuration options, including the option to automatically generate SBOM and scanning by Trivy.

For information on configuring a project, see the [Working with Projects](https://goharbor.io/docs/2.11.0/working-with-projects/) in the Harbor documentation.

In a project, there is a PUSH Command option which will give you the command templates for pushing to the Project repositories from different clients.

## Repositories

Each repository in a project has a COPY PULL Command option once an image/artifact has been selected for Docker and Podman.

Clicking on a tag in a repository will open up the information on the artifact, this can include an overview of the image, vulnerability summary, SBOM and build history.

## Using from the Command Line (Docker)

Important: Run these commands on a system that has Docker installed and has access to the EIDF Registry.

To login to the registry from a Docker client, you should use the following:

```bash
docker login registry.eidf.ac.uk
```

If you have previously done this and the credentials are valid, then it will confirm your login.

If you have not logged in, the following prompt will appear:

```bash
Username:
```

You should enter the username (your SAFE preferred username) which appears in the User profile from the web page.

```bash
Password:
```

You should copy the CLI Secret from the User Profile into this prompt.

The following should appear, unless you are using a credential helper, which will remove the warning.

```bash
WARNING! Your password will be stored unencrypted in <home directory>/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

From your command line, you can now push and pull images to the registry.

## Kubernetes/GPU Service Access

To pull images from the registry, from private or authenticated projects, you will need to add a secret to the namespace you are using and reference it in your job definition. Note that user tokens have a limited validity period. If you are regularly using a repository from a project, it would be recommended to create a robot account with limited privileges, this is discussed below.

### Robot Accounts

This can done via the Web Interface for the registry in project you want to pull images from.

In the project, choose the Robot Account tab, and create a new Robot Account.

Give your robot a name, a description to help you remember what it is for and set an expiry time in number of days. If you do not want to have the token expire, set it to -1.

Click Next and you should give the robot READ and PULL permissions on the repository. If you are looking to do more complicated workflows you may need to assign more permissions.

This will generate a robot user and secret. Take a note of these, the secret will not be shown again.

In the following sections, if you want to use a robot account, replace references to your username and CLI secret with the robot name and secret.

### Secret Creation when logged into via Docker config.json

Important: Run these commands on a system that has Docker, kubectl installed and has access to the EIDF Registry.

```bash
kubectl create secret generic eidfreg \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson -n <your namespace>
```

Running this will create a secret eidfreg in your project namespace. This can be used for Kubernetes to pull from your private repositories.

### YAML File Creation

Important: Run these commands on a system that has kubectl installed and has access to the EIDF Registry.

Encode your authorisation information into base64 for the secret:

```bash
echo '{"auths":{"https://registry.eidf.ac.uk":{"username":"<your username>","password":"<your CLI Secret>"}}}' | base64
```

This will generate a string of characters, it may have line breaks on output, and for the next part, you can remove the line breaks and make it a single line.

```bash
eyJhdXRocyI6eyJodHRwczovL3JlZ2lzdHJ5LmVpZGYuYWMudWsiOnsidXNlcm5hbWUiOiJyb2Jv
dCR0ZXN0X2xpYnJhcnkrZ3B1X2RlZmF1bHQiLCJwYXNzd29yZCI6InZrakRHRzNwSG9lTVc0bEtp
THRTZGt3ZXdlcVlpRDFxc1RPT2cifX19Cg==
```

Create a YAML File with the following information:

```bash
apiVersion: v1
kind: Secret
metadata:
  name: <replace with your secret name>
  namespace: <your project namespace>
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: >-
     <the output from the encoding process>
```

For example:

```bash
apiVersion: v1
kind: Secret
metadata:
  name: eidfreg
  namespace: eidf000ns
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: >-
    eyJhdXRocyI6eyJodHRwczovL3JlZ2lzdHJ5LmVpZGYuYWMudWsiOnsidXNlcm5hbWUiOiJyb2JvdCR0ZXN0X2xpYnJhcnkrZ3B1X2RlZmF1bHQiLCJwYXNzd29yZCI6InZrakRHRzNwSG9lTVc0bEtpTHRTZGt3ZXdlcVlpRDFxc1RPT2cifX19Cg==
```

To create the secret run:

```bash
kubectl apply -f <your filename> -n <your namespace>
```

### Using in a Job

Use the image name from the pull command for the repository. For example, this could be "registry.eidf.ac.uk/library/cuda-nbody:latest".

This is an example using an image and secret to access the registry. Replace the appropriate values with your own configuration.

The below example will function without a secret as the image is in a public repository.

``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  generateName: jobtest-
  labels:
    kueue.x-k8s.io/queue-name: <project-namespace>-user-queue
spec:
  completions: 1
  template:
    metadata:
      name: job-test
    spec:
      containers:
        - name: cudasample
          image: registry.eidf.ac.uk/library/cuda-nbody:latest
          args:
            - '-benchmark'
            - '-numbodies=512000'
            - '-fp64'
            - '-fullscreen'
          resources:
            requests:
              cpu: 2
              memory: 1Gi
            limits:
              cpu: 2
              memory: 4Gi
              nvidia.com/gpu: 1
      restartPolicy: Never
      imagePullSecrets:
        - name: <your secret>
```
