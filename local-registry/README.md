# docker-registry on centos 7

### lab model
```
------------+---------------------------+-------------
            |                           |
     ens33  |192.168.55.75       ens33  |192.168.55.76
+-----------+-----------+   +-----------+-----------+
| [ docker-reistry ]    |   | [ docker-node ]       |
|                       |   |                       |
|  docker               |   |     docker            |
|  docker registry      |   |                       |
|                       |   |                       |
|                       |   |                       |
+-----------------------+   +-----------+-----------+
```

### Step 1: Prepare all nodes
NOTE: This part should be done on all nodes of your cluster. Preparation step is very simple and anyone who works with Docker is very familiar with this process. All we need is to install latest Docker. This step simply follows official Docker documentation.
- First, edit hosts ```vi /etc/hosts```
    ```
    ### append

    192.168.55.75	docker-registry	dockerhub.gonapp.net
    192.168.55.76	docker-node
    ```
    Second, install required dependencies:

    ```
    $ sudo yum update
    $ sudo yum install -y yum-utils device-mapper-persistent-data lvm2
    ```

### Step 2: Add Docker repository & install:
- add repository
    ```
    # yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    ```
- Install Docker CE:
    ```
    # yum install -y docker-ce
    ```
- Enable Docker start on boot and start daemon:
    ```
    # systemctl enable docker
    # systemctl start docker
    ```

### Step 3: Run a local registry on node docker-registry
- Use a command like the following to start the registry container:

    ```
    # docker run -d -p 5000:5000 --restart=always --name registry registry:2

    Unable to find image 'registry:2' locally
    2: Pulling from library/registry
    486039affc0a: Pull complete
    ba51a3b098e6: Pull complete
    8bb4c43d6c8e: Pull complete
    6f5f453e5f2d: Pull complete
    42bc10b72f42: Pull complete
    Digest: sha256:7d081088e4bfd632a88e3f3bcd9e007ef44a796fddfe3261407a3f9f04abe1e7
    Status: Downloaded newer image for registry:2
    f3446bc82a2d7f5f4603531a40f5f6191ead7b3be0cbfd5a80ab60a5c0956c97
    ```
    check container state
    ```
    # docker ps

    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
    f3446bc82a2d        registry:2          "/entrypoint.sh /etc…"   39 seconds ago      Up 38 seconds       0.0.0.0:5000->5000/tcp   registry

    ```

#### run with storage customization
- follow

    ```
    # docker container stop registry

    # docker container rm -v registry

    # docker run -d   -p 5000:5000   --restart=always   --name registry   -v /mnt/registry:/var/lib/registry   registry:2

    68f830b63b05412da23b77e4c776c849d0eac7abd21e8126365c87272e1a4cce
    ```

- Copy an image from Docker Hub to your registry
    You can pull an image from Docker Hub and push it to your registry. The following example pulls the ubuntu:16.04 image from Docker Hub and re-tags it as my-ubuntu, then pushes it to the local registry. Finally, the ubuntu:16.04 and my-ubuntu images are deleted locally and the my-ubuntu image is pulled from the local registry.

    Pull the ubuntu:16.04 image from Docker Hub.
    ```
    $ docker pull ubuntu:16.04
    ```
    Tag the image as localhost:5000/my-ubuntu. This creates an additional tag for the existing image. When the first part of the tag is a hostname and port, Docker interprets this as the location of a registry, when pushing.
    ```
    # docker tag ubuntu:16.04 localhost:5000/my-ubuntu
    ```
    Push the image to the local registry running at localhost:5000:
    ```
    # docker push localhost:5000/my-ubuntu
    ```
    Remove the locally-cached ubuntu:16.04 and localhost:5000/my-ubuntu images, so that you can test pulling the image from your registry. This does not remove the localhost:5000/my-ubuntu image from your registry.
    ```
    # docker image remove ubuntu:16.04
    # docker image remove localhost:5000/my-ubuntu
    ```
    Pull the localhost:5000/my-ubuntu image from your local registry.
    ```
    # docker pull localhost:5000/my-ubuntu
	```

- stop registry

    ```
    # docker container stop registry

    # docker container rm -v registry
    ```

### Step 4: Setup cert for registry

- on registry node

    ```
    # mkdir certs
    # [root@docker-registry ~]# openssl req \
	>  -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
	>  -x509 -days 365 -out certs/domain.crt

	Generating a 4096 bit RSA private key
    ...++
    ...............++
    writing new private key to 'certs/domain.key'
    -----
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [XX]:VN
    State or Province Name (full name) []:HaNoi
    Locality Name (eg, city) [Default City]:HaNoi
    Organization Name (eg, company) [Default Company Ltd]:Gonstack
    Organizational Unit Name (eg, section) []:Devops
    Common Name (eg, your name or your server's hostname) []:dockerhub.gonapp.net
    Email Address []:admin@gonstack.com
    ```

- Copy the domain.crt file to /etc/docker/certs.d/dockerhub.gonapp.net:5000/ca.crt

    ```
    # mkdir -p /etc/docker/certs.d/dockerhub.gonapp.net:5000
    # cp certs/domain.crt /etc/docker/certs.d/dockerhub.gonapp.net:5000/ca.crt
    ```
- Run the registry:

    ```
    # docker run -d \
      --restart=always \
      --name registry \
      -v `pwd`/certs:/certs \
      -v /mnt/registry:/var/lib/registry \
      -e REGISTRY_HTTP_ADDR=0.0.0.0:5000 \
      -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
      -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
      -p 5000:5000 \
      registry:2

    e021f4474c1b169cedcc58a17316475626e63516522400147b66e70d5f81c988
    ```
### Step 5: In Docker node

	#### Accept insecure registry (Self signed certificate)

- coppy cert for docker node
    ```
    # mkdir -p ~/.docker/certs.d/dockerhub.gonapp.net:5000/
    # scp root@dockerhub.gonapp.net:/etc/docker/certs.d/dockerhub.gonapp.net:5000/ca.crt ~/.docker/certs.d/dockerhub.gonapp.net:5000/ca.crt
    ```

- ### linux
	```vi /etc/docker/daemon.json```

    ```

    {
      "insecure-registries" : ["dockerhub.gonapp.net:5000"]
    }
    ```

- ### mac
	```vi ~/.docker/daemon.json```
    ```
    {
      "insecure-registries" : ["dockerhub.gonapp.net:5000"]
    }
    ```
- final

    ```
        systemctl restart docker
    ```

	push image to registry:
    ```
    # docker tag python-testgatco dockerhub.gonapp.net:5000/python-testgatco

    # docker push dockerhub.gonapp.net:5000/python-testgatco

    ```
    The push refers to repository [dockerhub.gonapp.net:5000/python-testgatco]
    55970f8bdf6e: Pushed
    a5a578e6406f: Pushed
    4038210c6b82: Pushed
    22455df42300: Pushed
    065421ab2879: Pushed
    b3fb4a3eba4a: Pushed
    1fa8778eb779: Pushed
    fa0c3f992cbd: Pushed
    ce6466f43b11: Pushed
    719d45669b35: Pushed
    3b10514a95be: Pushed
    latest: digest: sha256:1ae22c681ff45a4917d14a8c179b1eb716ab184f2426d8fbcee7cff358cb52dc size: 2637
    Check images in server
    ```
    # docker images
    REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
    minio/minio         latest              456e5e732c89        4 days ago          33.5MB
    ubuntu              16.04               7aa3602ab41e        8 days ago          115MB
    hello-world         latest              2cb0d9787c4d        3 weeks ago         1.85kB
    registry            2                   b2b03e9146e1        4 weeks ago         33.3MB
    ```
	Pull image from registry to workstation
    ```
    $ docker pull dockerhub.gonapp.net:5000/python-testgatco
    $ docker images
    dockerhub.gonapp.net:5000/python-testgatco
    ```








### Thanks you