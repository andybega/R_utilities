Dealing with servers
================

-   [Starting and interacting with server](#starting-and-interacting-with-server)
    -   [EC2 for Rstudio](#ec2-for-rstudio)
    -   [EC2 for Docker](#ec2-for-docker)
    -   [Digital Ocean for Docker](#digital-ocean-for-docker)
    -   [Setup credentials](#setup-credentials)
-   [Deploying data/environment to server](#deploying-dataenvironment-to-server)
    -   [Manually](#manually)
    -   [Use Docker](#use-docker)
        -   [Docker + copy source files and data](#docker-copy-source-files-and-data)
        -   [Docker + copy image](#docker-copy-image)
-   [Utilities for interacting with servers](#utilities-for-interacting-with-servers)
    -   [Run scripts in background](#run-scripts-in-background)
    -   [Logging with `futile.logger`](#logging-with-futile.logger)
    -   [Script notifications via Pushbullet](#script-notifications-via-pushbullet)
    -   [File transfer with S3](#file-transfer-with-s3)

Starting and interacting with server
------------------------------------

EC2 vs Digital Ocean: in terms of getting a server going and interacting with it, Digital Ocean is easier to setup than EC2.

For running RStudio in the cloud, EC2 has an AMI that already includes RStudio. For Digital Ocean, the options are either to install RStudio manually, or use docker with one of the RStudio containers.

### EC2 for Rstudio

Check for the image ID at <http://www.louisaslett.com/RStudio_AMI/>.

Expose port 80 for RStudio.

``` bash
PEM=/path/to/certificate.pem
PUBDNS=[public dns ec2]
ssh -i $PEM ec2-user@$PUBDNS

sudo passwd rstudio
```

### EC2 for Docker

Using the default EC2 Linux AMI.

In security settings expose TCP 8787 for RStudio inside Docker.

``` bash
PEM=/path/to/certificate.pem
PUBDNS=[public dns ec2]
ssh -i $PEM ec2-user@$PUBDNS

sudo yum update -y
sudo yum install -y docker
sudo service docker start
sudo usermod -a -G docker ec2-user
logout
ssh -i $PEM ec2-user@$PUBDNS
docker info
```

### Digital Ocean for Docker

### Setup credentials

GitHub. When pushing to a remote repo, it will still ask for GitHub username/password. It's possible to [cache those](https://help.github.com/articles/caching-your-github-password-in-git/).

``` bash
git config --global user.name "Andreas Beger"
git config --global user.email "foo@foo.com"
```

AWS S3. In R:

``` r
Sys.setenv("AWS_ACCESS_KEY_ID" = "mykey",
           "AWS_SECRET_ACCESS_KEY" = "mysecretkey")
```

Pushbullet:

``` r
library("RPushbullet")
pbSetup()
```

Deploying data/environment to server
------------------------------------

A server is spun up and running, what now? How to get neccessary source files and data onto it?

### Manually

This is what I initially did. Rstudio on the server. Use `rsync` or `scp` to copy everything over, then login to Rstudio on the server and proceed.

-   Con: need a script to setup R environment
-   Con: pain to sync changes back and forth
-   Pro: in theory easier, less steps required

### Use Docker

Instead of running Rstudio on the server, use docker to run a container with Rstudio on the server.

-   Pro: you can run the container and thus work in the same environment
-   Pro: easier to deploy on server since environment will be the same
-   Con: requires some extra steps (running the container on the server)
-   Con: getting data out of the container also requires some extra steps

Some general points for using docker:

1.  It is easier to build an image with access to needed source files and data than to later copy the source files and data into the container.
2.  For large files, one solution is to use an external storage location (S3, Dropbox, etc.) and sync the file from there.
3.  For images that should not be public/on Docker hub, one can manually push the files or the entire image.

#### Docker + copy source files and data

Manually deploy a folder and Dockerfile, then build the image on the server. This requires less copying than building an image locally and then sending.

``` bash
# on local
cd /path/to/folder
ssh -i $PEM ec2-user@$PUBDNS mkdir [dest dir]
rsync -rvz --progress -e "ssh -i $PEM" \
    ./ ec2-user@${PUBDNS}:/home/ec2-user/countmodels

# on remote
ssh -i $PEM ec2-user@$PUBDNS
cd [dest dir]

CONTAG=[container name]
docker build -t $CONTAG ./
docker images

USER=[user]
PW=[pw]
docker run -dp 8787:8787 -e ROOT=TRUE -e USER=$USER -e PASSWORD=$PW $CONTAG
```

#### Docker + copy image

Build the container image locally and manually transfer it to the server, e.g. using `rsync` or `scp`. This can be slow if the container is several GB in size, like the `rocker/tidyverse` container is.

Get a EC2 or DO server running using the instructions above. Then:

On local:

``` bash
cd /path/to/Dockerfile
CONTAG=[container name]
docker build -t $CONTAG ./

docker save $CONTAG > ${CONTAG}.tar
rsync -vz --progress -e "ssh -i $PEM" \
    ./${CONTAG}.tar ec2-user@${PUBDNS}:/home/ec2-user
```

On destination:

``` bash
CONTAG=[container name]
docker load < ${CONTAG}.tar
docker images

USER=[user]
PW=[pw]
docker run -dp 8787:8787 -e ROOT=TRUE -e USER=$USER -e PASSWORD=$PW $CONTAG
```

This is maybe good for smaller container. For larger images, probably easier to sync the files and build the image on the server from the Dockerfile. This approach uses more space, as the `.tar` file will be around while docker is extracting the image.

Utilities for interacting with servers
--------------------------------------

Helper stuff for running stuff on a server and getting files off of it.

### Run scripts in background

Run scripts in background rather than interactive session so they keep running. This obviates the need for something like `tmux`.

``` bash
cd /my/project
nohup Rsript myproject.R > myproject.log 2>&1 &

# check status
tail myproject.log
```

### Logging with `futile.logger`

When running scripts that take a while to run, and/or which run on a server somewhere, it is nice to have some measure of progress. Something like this:

``` r
t0 <- proc.time()
Sys.sleep(2)
cat(sprintf("%s: something happened\n", Sys.time()))
cat(sprintf("Script finished in %ss\n", round((proc.time() - t0)["elapsed"])))
```

    ## 2017-10-20 14:43:25: something happened
    ## Script finished in 2s

works but is a bit unwieldy. Need to put newlines ("") at each `cat()` and what to do if you suddenly don't want log output. Instead:

``` r
library("futile.logger")
flog.info("Hello")
flog.error("Something bad happened, but we keep going")
```

    ## INFO [2017-10-20 14:43:25] Hello
    ## ERROR [2017-10-20 14:43:25] Something bad happened, but we keep going

### Script notifications via Pushbullet

When is that script running on a server done?

1.  Connect to server and check periodically.
2.  Get notifications when the script is done (or errors out).

There seem to be several ways of doing this in R. Pushbullet via [RPushbullet](https://github.com/eddelbuettel/rpushbullet) is one option. It's free, and was easy to setup.

``` r

library("RPushbullet")

# 1st time setup:
if (!file.exists("~/.rpushbullet.json") & interactive()) {
  api_key <- readline(prompt = "Enter Pushbullet API key: ")
  dev <- pbGetDevices(api_key)
  jsonlite::toJSON(list(key = api_key,
                        devices = dev$devices$iden,
                        names = dev$devices$nickname),
                   pretty = TRUE, auto_unbox = TRUE) %>%
    cat(., file = "~/.rpushbullet.json")
} else if (!file.exists("~/.rpushbullet.json")) {
  stop("Could not find RPushbullet config file; run interactively to setup")
}

# To send notifications
pbPost("note", title = "Server: something happened", body = sprintf("Finished %s", 2))


# This will send any errors as notifications as well; not good to have this on
# during an interactive session while debugging.
options(error = function() {
  library("RPushbullet")
  pbPost("note", "Server: Error", geterrmessage())
  if(!interactive()) stop(geterrmessage())
})
  
```

### File transfer with S3

Use an Amazon S3 bucket to store and access files. I created an IAM user with access to only a data store single bucket for this, since the credentials for that user might potentially live on a server.

Andrew Heiss has some code that uses `s3mpi`, which has some core functions for writing and reading R objects with S3 as the go between. It seems that the more general `aws.s3` packages provides the same functionality but with a more exhaustive feature set.

``` r
library("aws.s3")
Sys.setenv("AWS_ACCESS_KEY_ID" = "mykey",
           "AWS_SECRET_ACCESS_KEY" = "mysecretkey")

# this should work
bucketlist()

get_bucket("my-bucket")
```

There are several different ways to read/write from/to S3, but some basic ones:

-   `s3save()`/`s3load()` like `save()`/`load()`
-   `s3saveRDS()`/`s3loadRDS()` like `saveRDS()`/`loadRDS()`
-   for files on disk use `save_object()` and `put_object()` to read and write local files to S3
    -   E.g. to save a CSV to S3 one would: `write_csv()` -&gt; `put_object()`
    -   for smaller files [maybe also](https://github.com/cloudyr/aws.s3/issues/170): `s3read_using(FUN = read.csv, object = "s3://myBucketName/aFolder/fileName.csv")`

To get results from S3:

``` f
save_object(object = "/path/in/bucket/object.rda",
            bucket = "mybucket",
            file = "/path/on/local")
```