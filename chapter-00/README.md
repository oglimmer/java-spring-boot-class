# Chapter 00

Install java, an IDE/Editor and docker

# Goal

You have Java and docker installed.

# Context and Knowledge

* Java should be downwards compatible, so the latest version should work. **should** ;)
* Java comes as a JDK or a JRE
* JDK is the Java Development Kit
* JRE is the Java Runtime Environment
* All JDKs include a JRE - so you want to install a JDK
* Java is developed here : https://openjdk.org/ - but due to Oracle's licence model you need to install a "Java Distribution"
* the easiest way is to just install it via packet manager - alternatively you can use any distro and use it. Never install "OpenJDK from openjdk.org page" or any "Oracle Java"!
    * debian: `apt install openjdk-17-jdk` or https://adoptium.net/temurin/releases/ for a latest version of Java
    * macos: `brew install openjdk` (this installs the latest version)
    * windows: `winget install EclipseAdoptium.Temurin.21.JDK`
* docker makes development in many situations much easier, as it easily provides 3rd party software like webservers or databases


# Step 1

Install a JDK and either VS Code or IntelliJ (community should be good enough).

# Step 2

You should be able to use java (the runtime engine is called java)

```bash
java --version
```

You should be able to use the java compiler

```bash
javac --version
```

But nobody uses the javac directly. Nobody. Always use maven or gradle. We'll see this in the next chapter.

# Step 3 

Download docker desktop from https://www.docker.com/products/docker-desktop/ and install it.

Docker is very easy to use and doesn't require any addition configuration.

After installing it, here are first steps you might want to try:

```bash
# we start a nginx webserver and map port 80 from inside the container to your host
docker run --rm -p 80:80 nginx
# now try http://localhost/ in your browser
```

or

```bash
# start a container which has curl and jq installed, do an GET http request to math.oglimmer.com to solve 5+7*4 and format the resulting JSON nicely
docker run --rm apteno/alpine-jq /bin/sh -c 'curl "https://math.oglimmer.de/v1/calc?expression=5+7*4" -s | jq'
```

or

```bash
# start a MariaDB database with user=root, passord=root
docker run --rm -e MARIADB_ROOT_PASSWORD=root -p 3306:3306 mariadb
# download any MariaDB (or Mysql) Client and access your database at localhost with root/root
```

# What we've learnt

* some java terminology
* docker
