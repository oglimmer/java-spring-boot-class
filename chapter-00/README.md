# Chapter 01

Install java and an IDE/Editor

# Goal

You have Java installed.

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
    * windows: `winget install Oracle.JDK.18`

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

# What we've learnt

* some java terminology 
