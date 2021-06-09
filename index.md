# Creating a private Maven repository

Sign in to your company based GitHub account, create a new private repository for the package. It is not possible to create a maven package without a git repository.

![new repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/maven-guide/create-repo.png)

This will create your repository, you can push your package as usual

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/maven-guide/new-repo.png)

Take a note of the repository url, this will be required later when setting up you local authorisation.

---

## If you do not have an existing API key for your github account then you will need to create one

First go to settings > Developer settings

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/maven-guide/api-gen-1.png)

Then Personal access tokens > Generate new token

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/maven-guide/api-gen-2.png)

Click ```write:packages``` which will provide the necessary default privileges, add a Note as a reminder of what the token is being used for.

![repo](https://s3.eu-west-1.amazonaws.com/ovalgeneric/public/assets/maven-guide/api-gen-3.png)

After the token generates copy it for use later.

---

## Setting up Maven locally

In your home directory there should be a ```.m2``` directory. In here is there is a ```settings.xml``` open it otherwise create it. This is where you will keep the configuration setting for github access.

The file needs a _profiles_ section for each repository you want to pull packages from. here we add both the maven.org repository and our new github repository, note the url contains the repository url we saved previously with the added subdomain of __maven.pkg__ and suffix __.git__

Then we add a _servers_ section where we add the authentication for our private repository, the password property is where we paste your GitHub API key.

```XML
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <activeProfiles>
    <activeProfile>github</activeProfile>
  </activeProfiles>

  <profiles>
    <profile>
      <id>github</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2</url>
          <releases><enabled>true</enabled></releases>
          <snapshots><enabled>true</enabled></snapshots>
        </repository>
        <repository>
          <id>github</id>
          <name>OvalUK</name>
          <url>https://maven.pkg.github.com/ovaluk/maven-test.git</url>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <servers>
    <server>
      <id>github</id>
      <username>alexwhiteoval</username>
      <password>--- YOUR TOKEN GOES HERE ---</password>
    </server>
  </servers>
</settings>
```

Now that this is setup your private repository is now available to all your local maven projects. You can create new packages and import those into apps without the need to input your API again.

---

## Setting up a package to push to your github repository

Now we can edit the package ```pom.xml``` so this when you deploy it will know where to put the package.

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.uk.ovalbusinesssolutions</groupId>
    <artifactId>maven-test</artifactId>
    <version>1.0-SNAPSHOT</version>
   
    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <distributionManagement>
        <repository>
            <id>github</id>
            <name>OvalUK</name>
            <url>https://maven.pkg.github.com/OvalUK/maven-test</url>
        </repository>
    </distributionManagement>

</project>
```

Here we have added a _distributionManagement_ property where we list our github repository, most of this data is the same the ```settings.xml``` except the _url_ which does not include the __.git__ suffix.

with all this setup we can ```git push ...```our repository and ```mvn deploy``` to publish the package to our private repository.

### Automatic deployments with GitHub

If you using GitHub actions to complete the build and deployment process, the source repository needs to have a actions yaml file, name the file appropriately in the following location ```./.github/workflows/MY_ACTION.yml``` The contents should be as follows:

```YAML
name: Maven package deployment

on:
  push:
    tags:
      - "*"
  release:
    types:
      - published

jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-20.04
    steps:
    - uses: actions/checkout@v2
    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8
    - name: Build with Maven
      run: |
        arrTag=(${GITHUB_REF//\// })
        VERSION="${arrTag[2]}"
        VERSION="${VERSION//v}"
        mvn --batch-mode release:update-versions -DdevelopmentVersion=$VERSION
        mvn -B package --file pom.xml
    - name: Deploy package
      env:
        GITHUB_USER: ${{ "{{ github.actor " }}}}
        GITHUB_TOKEN: ${{ "{{ secrets.GITHUB_TOKEN " }}}}
      run: |
        echo "<settings><servers><server><id>github</id><username>${GITHUB_USER}</username><password>${GITHUB_TOKEN}</password></server></servers></settings>" > ~/.m2/settings.xml
        mvn deploy
```

This above action will build the package using the ubuntu-20.04 operating system with Java 1.8 installed. during the build phase the version number will be updated using the tag value. Git tags should be in the format _v#.#_ or _v#.#-SNAPSHOT_ this will result in a dependency similar to:

```XML
<dependency>
  <groupId>com.uk.ovalbusinesssolutions</groupId>
  <artifactId>maven-test</artifactId>
  <version>0.4-SNAPSHOT</version>
</dependency>
```

The deploy phase takes the built package and deploys it to our private maven repository using the deploy command.

---

## Pulling the package into a project

To pull a package into a new or existing project you ```pom.xml```will need a _repositories_ property with each github repository you need to use and then the usual _dependency_ property

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.uk.ovalbusinesssolutions</groupId>
    <artifactId>maven-echo-testing</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <repositories>
        <repository>
            <id>github</id>
            <url>https://maven.pkg.github.com/OvalUK/maven-test</url>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>com.uk.ovalbusinesssolutions</groupId>
            <artifactId>maven-test</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>

</project>
```
