
เอกสารฉบับนี้ ได้รวบรวมคำแนะนำการใช้งานและวิธีการในการสร้าง image อย่างมีประสิทธิภาพ

Docker จะสร้าง image โดยอัตโนมัติ ด้วยการอ่านคำชุดคำสั่งจาก `Dockerfile` ( ไฟล์ที่รวบรวมชุดคำสั่งทั้งหมดที่จำเป็นต่อการสร้าง Image โดยเรียงลำดับจากบนลงล่าง )
การเขียน `Dockerfile` นั้น จึงจำเป็นต้องอ้างอิงจากเอกสาร เนื่องจากมีชุดคำสั่งที่เฉพาะเจาะจง คุณสามารถดูได้จาก [Dockerfile reference](../../engine/reference/builder.md)

Docker image ประกอบไปด้วย ชั้นข้อมูล read-only โดยที่แต่ละชั้นถูกสร้างโดยคำสั่งต่างๆ ที่อยู่ใน Dockerfile 

ชั้นข้อมูลจะถูกวางจากล่างขึ้นบน (Stack) โดยแต่ละชั้นคือการเปลี่ยนแปลงข้อมูลเพิ่มเติมจากชั้นด้านล่าง ขึ้นมาเรื่อย ๆ

พิจารณา `Dockerfile` นี้:

```dockerfile
FROM ubuntu:18.04
COPY . /app
RUN make /app
CMD python /app/app.py
```

คำสั่งแต่ละบรรทัด สร้างชั้นข้อมูลขึ้นมา:

- `FROM` สร้างชั้นข้อมูลจาก Image หลักที่ชื่อว่า `ubuntu:18.04`
- `COPY` เพิ่มไฟล์ทั้งหมดจากโฟล์เดอร์ปัจจุบัน ไปยังชั้นถัดไป 
- `RUN` build แอปพลิเคชัน ด้วยคำสั่ง `make`
- `CMD` ระบุคำสั่งที่จะใช้เมื่อนำ Image  มาสร้าง Container

เมื่อเราสร้าง Container จาก Image เราได้ทำงานเพิ่มชั้นข้อมูล Writable อีกชั้นหนึ่ง (the "container layer")  ทุกการเปลี่ยนแปลงที่เกิดขึ้นกับ Container นี้นั้น เช่น การเพิ่มไฟล์ใหม่ การเปลี่ยนแปลงข้อมูลที่มีอยู่แล้ว หรือการลบไฟล์ จะถูกทำบนชั้น Writable นี้

เรียนรู้เรื่อง ชั้นต่างๆ ของ Image ได้ที่ [About storage drivers](../../storage/storagedriver/index.md).

## General guidelines and recommendations

### Create ephemeral containers

Imageที่กำหนดโดยDockerfile ของคุณควรสร้าง ephemeral containers ที่เป็นไปได้
โดย "ephemeral" หมายความว่า containerสามารถหยุด และทำลาย จากนั้นก็สร้างใหม่และแทนที่ ด้วยการตั้งค่าและการกำหนดค่าขั้นต่ำ

อ้างถึงกระบวนการ(https://12factor.net/processes)ภายใต้ The Twelve-factor App เพื่อให้รู้สึกถึงแรงจูงใจ
ในการใช้containerในรูปแบบที่ไม่จดจำสถานะ


### Understand build context

เมื่อคุณใช้คำสั่ง สร้าง Docker, directory ที่กำลังทำงานจะถูกเรียกว่า build context โดยเริ่มต้น Dockerfile จะอยู่ที่นี่ แต่คุณ
สามารถระบุตำแหน่งอื่นได้ด้วย file flag (-f) เนื้อหาแบบเรียกซ้ำทั้งหมดของไฟล์และ directories ใน
directoty ปัจจุบัน จะถูกส่งไปยัง Docker daemon เป็น build context.


> Build context example
>
> สร้าง directory สำหรับ build context แล้วใช้ cd เพื่อเข้าไปเขียน "Hello"ลงในไฟล์ Hello และสร้าง
> Dockerfile ที่รัน cat สร้าง image จาก build context (.): 
> 
>
> ```shell
> mkdir myproject && cd myproject
> echo "hello" > hello
> echo -e "FROM busybox\nCOPY /hello /\nRUN cat /hello" > Dockerfile
> docker build -t helloapp:v1 .
> ```
> ย้าย Dockerfile และ hello ไปยัง directories ที่แยกต่างหาก และสร้าง image
>เวอร์ชันที่สอง(โดยไม่ต้องพึ่งพาcacheจากการสร้างล่าสุด)  
>ใช้ -f เพื่อระบุตำแหน่งของ Dockerfile และ directory ของ  build context: 
> 
>
> ```shell
> mkdir -p dockerfiles context
> mv Dockerfile dockerfiles && mv hello context
> docker build --no-cache -t helloapp:v2 -f dockerfiles/Dockerfile context
> ```

การรวมไฟล์ที่ไม่จำเป็นสำหรับการสร้าง image ส่งผลให้เกิดการ build context และ image ที่มีขนาดใหญ่ สิ่งนี้จะเพิ่มเวลาในการสร้างimage, 
เวลาที่จะpull และ push และเพิ่มขนาดruntime ของcontainer หากต้องการดูว่า 
build context ของคุณมีความใหญ่ขนาดไหนให้ค้นหาโดยใช้ข้อความนี้ตอนที่คุณกำลังสร้าง Dockerfile :



```none
Sending build context to Docker daemon  187.8MB
```

### Pipe Dockerfile through `stdin`

Docker has the ability to build images by piping `Dockerfile` through `stdin`
with a _local or remote build context_. Piping a `Dockerfile` through `stdin`
can be useful to perform one-off builds without writing a Dockerfile to disk,
or in situations where the `Dockerfile` is generated, and should not persist
afterwards.

> The examples in this section use [here documents](http://tldp.org/LDP/abs/html/here-docs.html)
> for convenience, but any method to provide the `Dockerfile` on `stdin` can be
> used.
> 
> For example, the following commands are equivalent: 
> 
> ```bash
> echo -e 'FROM busybox\nRUN echo "hello world"' | docker build -
> ```
> 
> ```bash
> docker build -<<EOF
> FROM busybox
> RUN echo "hello world"
> EOF
> ```
> 
> You can substitute the examples with your preferred approach, or the approach
> that best fits your use-case.


#### Build an image using a Dockerfile from stdin, without sending build context

Use this syntax to build an image using a `Dockerfile` from `stdin`, without
sending additional files as build context. The hyphen (`-`) takes the position
of the `PATH`, and instructs Docker to read the build context (which only
contains a `Dockerfile`) from `stdin` instead of a directory:

```bash
docker build [OPTIONS] -
```

The following example builds an image using a `Dockerfile` that is passed through
`stdin`. No files are sent as build context to the daemon.

```bash
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```

Omitting the build context can be useful in situations where your `Dockerfile`
does not require files to be copied into the image, and improves the build-speed,
as no files are sent to the daemon.

If you want to improve the build-speed by excluding _some_ files from the build-
context, refer to [exclude with .dockerignore](#exclude-with-dockerignore).

> **Note**: Attempting to build a Dockerfile that uses `COPY` or `ADD` will fail
> if this syntax is used. The following example illustrates this:
> 
> ```bash
> # create a directory to work in
> mkdir example
> cd example
> 
> # create an example file
> touch somefile.txt
> 
> docker build -t myimage:latest -<<EOF
> FROM busybox
> COPY somefile.txt .
> RUN cat /somefile.txt
> EOF
> 
> # observe that the build fails
> ...
> Step 2/3 : COPY somefile.txt .
> COPY failed: stat /var/lib/docker/tmp/docker-builder249218248/somefile.txt: no such file or directory
> ```

#### Build from a local build context, using a Dockerfile from stdin

Use this syntax to build an image using files on your local filesystem, but using
a `Dockerfile` from `stdin`. The syntax uses the `-f` (or `--file`) option to
specify the `Dockerfile` to use, using a hyphen (`-`) as filename to instruct
Docker to read the `Dockerfile` from `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

The example below uses the current directory (`.`) as the build context, and builds
an image using a `Dockerfile` that is passed through `stdin` using a [here
document](http://tldp.org/LDP/abs/html/here-docs.html).

```bash
# create a directory to work in
mkdir example
cd example

# create an example file
touch somefile.txt

# build an image using the current directory as context, and a Dockerfile passed through stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF
```

#### Build from a remote build context, using a Dockerfile from stdin

Use this syntax to build an image using files from a remote `git` repository, 
using a `Dockerfile` from `stdin`. The syntax uses the `-f` (or `--file`) option to
specify the `Dockerfile` to use, using a hyphen (`-`) as filename to instruct
Docker to read the `Dockerfile` from `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

This syntax can be useful in situations where you want to build an image from a
repository that does not contain a `Dockerfile`, or if you want to build with a custom
`Dockerfile`, without maintaining your own fork of the repository.

The example below builds an image using a `Dockerfile` from `stdin`, and adds
the `hello.c` file from the ["hello-world" Git repository on GitHub](https://github.com/docker-library/hello-world).

```bash
docker build -t myimage:latest -f- https://github.com/docker-library/hello-world.git <<EOF
FROM busybox
COPY hello.c .
EOF
```

> **Under the hood**
>
> When building an image using a remote Git repository as build context, Docker 
> performs a `git clone` of the repository on the local machine, and sends
> those files as build context to the daemon. This feature requires `git` to be
> installed on the host where you run the `docker build` command.

### Exclude with .dockerignore

To exclude files not relevant to the build (without restructuring your source
repository) use a `.dockerignore` file. This file supports exclusion patterns
similar to `.gitignore` files. For information on creating one, see the
[.dockerignore file](../../engine/reference/builder.md#dockerignore-file).

### Use multi-stage builds

[Multi-stage builds](multistage-build.md) allow you to drastically reduce the
size of your final image, without struggling to reduce the number of intermediate
layers and files.

Because an image is built during the final stage of the build process, you can
minimize image layers by [leveraging build cache](#leverage-build-cache).

For example, if your build contains several layers, you can order them from the
less frequently changed (to ensure the build cache is reusable) to the more
frequently changed:

* Install tools you need to build your application

* Install or update library dependencies

* Generate your application

A Dockerfile for a Go application could look like:

```dockerfile
FROM golang:1.11-alpine AS build

# Install tools required for project
# Run `docker build --no-cache .` to update dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies with Gopkg.toml and Gopkg.lock
# These layers are only re-built when Gopkg files are updated
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# Install library dependencies
RUN dep ensure -vendor-only

# Copy the entire project and build it
# This layer is rebuilt when a file changes in the project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# This results in a single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```

### Don't install unnecessary packages

To reduce complexity, dependencies, file sizes, and build times, avoid
installing extra or unnecessary packages just because they might be "nice to
have." For example, you don’t need to include a text editor in a database image.

### Decouple applications

Each container should have only one concern. Decoupling applications into
multiple containers makes it easier to scale horizontally and reuse containers.
For instance, a web application stack might consist of three separate
containers, each with its own unique image, to manage the web application,
database, and an in-memory cache in a decoupled manner.

Limiting each container to one process is a good rule of thumb, but it is not a
hard and fast rule. For example, not only can containers be
[spawned with an init process](../../engine/reference/run.md#specify-an-init-process),
some programs might spawn additional processes of their own accord. For
instance, [Celery](http://www.celeryproject.org/) can spawn multiple worker
processes, and [Apache](https://httpd.apache.org/) can create one process per
request.

Use your best judgment to keep containers as clean and modular as possible. If
containers depend on each other, you can use [Docker container networks](../../network/index.md)
to ensure that these containers can communicate.

### Minimize the number of layers

In older versions of Docker, it was important that you minimized the number of
layers in your images to ensure they were performant. The following features
were added to reduce this limitation:

- Only the instructions `RUN`, `COPY`, `ADD` create layers. Other instructions
  create temporary intermediate images, and do not increase the size of the build.

- Where possible, use [multi-stage builds](multistage-build.md), and only copy
  the artifacts you need into the final image. This allows you to include tools
  and debug information in your intermediate build stages without increasing the
  size of the final image.

### Sort multi-line arguments

Whenever possible, ease later changes by sorting multi-line arguments
alphanumerically. This helps to avoid duplication of packages and make the
list much easier to update. This also makes PRs a lot easier to read and
review. Adding a space before a backslash (`\`) helps as well.

Here’s an example from the [`buildpack-deps` image](https://github.com/docker-library/buildpack-deps):

```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

### Leverage build cache

When building an image, Docker steps through the instructions in your
`Dockerfile`, executing each in the order specified. As each instruction is
examined, Docker looks for an existing image in its cache that it can reuse,
rather than creating a new (duplicate) image.

If you do not want to use the cache at all, you can use the `--no-cache=true`
option on the `docker build` command. However, if you do let Docker use its
cache, it is important to understand when it can, and cannot, find a matching
image. The basic rules that Docker follows are outlined below:

- Starting with a parent image that is already in the cache, the next
  instruction is compared against all child images derived from that base
  image to see if one of them was built using the exact same instruction. If
  not, the cache is invalidated.

- In most cases, simply comparing the instruction in the `Dockerfile` with one
  of the child images is sufficient. However, certain instructions require more
  examination and explanation.

- For the `ADD` and `COPY` instructions, the contents of the file(s)
  in the image are examined and a checksum is calculated for each file.
  The last-modified and last-accessed times of the file(s) are not considered in
  these checksums. During the cache lookup, the checksum is compared against the
  checksum in the existing images. If anything has changed in the file(s), such
  as the contents and metadata, then the cache is invalidated.

- Aside from the `ADD` and `COPY` commands, cache checking does not look at the
  files in the container to determine a cache match. For example, when processing
  a `RUN apt-get -y update` command the files updated in the container
  are not examined to determine if a cache hit exists.  In that case just
  the command string itself is used to find a match.

Once the cache is invalidated, all subsequent `Dockerfile` commands generate new
images and the cache is not used.

## ชุดคำสั่งต่างๆ ใน Dockerfile

คำแนะนำต่อไปนี้ถูกออกแบบมาเพื่อช่วยให้คุณสร้าง `Dockerfile` ที่มีประสิทธิภาพและง่ายต่อการดูแล

### FROM

[Dockerfile reference for the FROM instruction](../../engine/reference/builder.md#from)

ทุกครั้งที่่เป็นไปได้ คุณควรใช้ Official Image เป็นฐาน Image หลักในการเขียน `Dockerfile` เราแนะนำ [Alpine image](https://hub.docker.com/_/alpine/) เนื่องจากว่า Image เหล่านี้มีการควบคุมให้มีขนาดเล็ก (น้อยกว่า 5 MB ) แต่ยังคงความเป็นระบบปฏิบัติการ Linux อย่างเต็มที่เหมือนเดิม

### LABEL

[Understanding object labels](../../config/labels-custom-metadata.md)

คุณสามารถติดป้าย (label) ให้กับ image ของคุณ  ไม่ว่าจะด้วยชื่อของ project หรือ ข้อมูลใบอนุญาต ทั้งนี้เพื่อการง่ายขึ้นสำหรับการทดสอบอัตโนมัติ หรือเพื่อเหตุผลอื่นๆ 

เพื่อเริ่มการติดป้าย เริ่มต้นบรรทัดใหม่ด้วยคำว่า `LABEL` และตามด้วย key-value ที่ต้องการ

ตัวอย่างดังต่อไปนี้ แสดงให้เห็นถึง รูปแบบที่ถูกต้องในการติดป้ายให้กับ Image 

> Strings ที่มีช่องว่างต้องอยู่ใน `" "` **หรือ** ต้องถูก escape (`/`) 
> 
> เครื่องหมาย `"` ที่อยู่ใน String ก็ต้องถูก escape ด้วยเช่นกัน

```dockerfile
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor1="ACME Incorporated"
LABEL vendor2=ZENITH\ Incorporated
LABEL com.example.release-date="2015-02-12"
LABEL com.example.version.is-production=""
```

Image หนึ่งสามารถมีได้มากกว่าหนึ่งป้าย 
ใน Docker เวอร์ชัน 1.10 นั้น ผู้ใช้งานถูกแนะนำให้รวมทุกป้ายเข้าด้วยกันเป็นอันเดียว ในหนึ่งคำสั่ง `LABEL` เพื่อป้องกันการสร้างชั้นข้อมูลหลายชั้น ปัญหานี้ถูกแก้ไขแล้วในเวอร์ชันปัจจุบัน แต่การใช้งานแบบเก่าก็ยังคงทำได้อยู่

```dockerfile
# การติดหลายๆป้ายด้วยคำสั่ง LABEL เพียงครั้งเดียว
LABEL com.example.version="0.0.1-beta" com.example.release-date="2015-02-12"
```

การเขียนแบบด้านบนสามารถ เขียนได้อีกแบบคือ

```dockerfile
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-beta= \
      com.example.is-production="" \
      com.example.version="0.0.1-beta" \
      com.example.release-date="2015-02-12"
```

ดู [Understanding object labels](../../config/labels-custom-metadata.md)
เพื่อการศึกษาเพิ่มเรื่อง Key-value 

### RUN

[เอกสารเพิ่มเติมสำหรับ ชุดคำสั่ง RUN ](../../engine/reference/builder.md#run)

การแบ่งคำสั่ง `RUN` เป็นหลายๆบรรทัดด้วยเครื่องหมาย `/` (Backslashe) จะทำให้ `Dockerfile` อ่านสะดวกและง่ายต่อการทำความเข้าใจมากขึ้น 

#### apt-get

สื่งที่น่าจะพบได้บ่อยที่สุดในชุดคำสั่ง `RUN` คือ คำสั่ง `apt-get`เนื่องจากมันถูกใช้เพื่อลง package ต่างๆ เพราะฉะนั้นคำสั่ง `RUN apt-get` จึงมีข้อแนะนำและข้อจำกัดบางอย่างที่ควรรู้ 

ควรหลีกเลี่ยงการเขียน `RUN apt-get upgrade` และ `dist-upgrade` เนื่องจากมี package จำนวนมากใน image parent ที่ไม่สามารถถูก upgrade ภายใน 
[unprivileged container](../../engine/reference/run.md#security-configuration) ได้ 

ถ้าคุณมี package ใดๆใน parent image ที่เป็นของเวอร์ชันเก่า ให้คุณติดต่อผู้ดูแล image นั้นๆ เพื่อดำเนินการ upgrade แต่ถ้าคุณมี package ที่ต้องการการอัพเดท คุณสามารถใช้คำสั่ง   `apt-get install -y <ชื่อ package>` โดยตรงได้ทันที

ควรใช้คำสั่ง `RUN apt-get update` และ `apt-get install` ภายในชุดคำสั่ง `RUN` เดียวกันเสมอ เช่น

```dockerfile
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo
```

การใช้คำสั่ง `apt-get update` เพียงอย่างเดียวในชุดคำสั่ง `RUN` นั้นจะสร้าง cache ไว้ใน Docker และทำให้คำสั่ง `apt-get install` ต่อมา ล้มเหลว

ยกตัวอย่างเช่น 
Dockerfile นี้

```dockerfile
FROM u buntu:18.04
RUN apt-get update
RUN apt-get install -y curl
```

หลังจากที่เราสร้าง image นี้แล้ว  ทุกชั้นข้อมูลจะถูกเก็บไว้ที่ cache ของ Docker  
สมมุติว่าเวลาต่อมา เราปรับเปลี่ยน `apt-get install` ด้วยการเพิ่ม package อื่นเช่น

```dockerfile
FROM ubuntu:18.04
RUN apt-get update
RUN apt-get install -y curl nginx
```

Docker จะถือว่าทั้งสองไฟล์นั้นเหมือนกัน และทำการใช้ข้อมูล cache จากครั้งก่อน ทำให้ `apt-get update` _ไม่ถูก_ จัดการ 

เมื่อเป็นเช่นนั้นแล้ว ในการสร้าง image ครั้งนี้ package `curl` และ `nginx` อาจจะไม่ได้ถูกอัปเดต

การใช้คำสั่ง `RUN apt-get update && apt-get install -y` ทำให้เรามั่นใจได้ว่า Docker จะติดตั้ง package เวอร์ชันล่าสุดเสมอ โดยไม่ต้องเขียนโปรแกรมอะไรเพิ่มเติม  เทคนิคนี้ีชื่อว่า "cache busting" คุณสามารถใช้วิธีอื่นเพื่อให้ได้ผลลัพท์เหมือนที่กล่าวมาข้างต้น ด้วยการระบุ เวอร์ชันของ package ที่ต้องการใช้งานโดยตรง ซึ่งวิธีนี้ถูกเรียกว่า version pinning

ยกตัวอย่างเช่น:

```dockerfile
RUN apt-get update && apt-get install -y \
    package-bar \
    package-baz \
    package-foo=1.3.*
```

การระบุเวอร์ชันโดยตรง (Version pinning) บังคับ Docker ให้สร้าง Image จากการดึงจากเวอร์ชันนั้นๆ จาก Internet โดยไม่สนใจว่า package เวอร์ชันนั้นมีอยู่ใน cache หรือไม่ 

เทคนิคนี้ยังคงช่วยลดความผิดพลาดที่เกิดขึ้นจากการเปลี่ยนแปลงที่ไม่คาดคิดใน package เหล่านั้นอีกด้วย 

ตัวอย่างด้านล่างเหล่านี้ คือ format ที่ถูกต้องในการเขียนชุดคำสั่ง `RUN` ซึ่งถูกแนะนำให้ใช้งานคู่กับ `apt-get` 


```dockerfile
RUN apt-get update && apt-get install -y \
    aufs-tools \
    automake \
    build-essential \
    curl \
    dpkg-sig \
    libcap-dev \
    libsqlite3-dev \
    mercurial \
    reprepro \
    ruby1.9.1 \
    ruby1.9.1-dev \
    s3cmd=1.1.* \
 && rm -rf /var/lib/apt/lists/*
```

`s3cmd` ระบุเวอร์ชันเป็น `1.1.*`  ถ้า image ก่อนหน้านี้ใช้เวอร์ชันเก่ากว่านี้ การระบุเวอร์ชันใหม่จะทำให้ไม่เกิดการใช้ cache ใน `apt-get update` และทำให้มั่นใจได้ว่า Docker จะติดตั้ง package เวอร์ชันใหม่เสมอ 

การเขียน package แบบแยกบรรทัดแบบนี้ จะทำให้ความผิดพลาดที่จะเขียนชื่อ package ซ้ำกันน้อยลง 

เมื่อคุณล้าง apt cache โดยการลบ `/var/lib/apt/lists` นั้น ขนาดของ image จะเล็กลง เนื่องจาก apt cache จะไม่ถูกจัดเก็บในชั้นข้อมูลอีกต่อไป

เมื่อชุดคำสั่ง `RUN` เริ่มด้วย `apt-get update` ข้อมูล cache ของ package จะถูกรีเซทใหม่อยู่เสมอ ก่อนที่จะทำคำสั่ง `apt-get install`

> Debian และ Ubuntu official image ทำ [`apt-get clean โดยอัตโนมัติ`](https://github.com/moby/moby/blob/03e2923e42446dbb830c654d0eec323a0b4ef02a/contrib/mkimage/debootstrap#L82-L105) 
> จึงไม่จำเป็นต้องระบุในชุดคำสั่ง `RUN` ก็ได้

#### การใช้ pipes

ชุดคำสั่ง `RUN` บางครั้งสามารถใช้ pipe (`|`) ในการส่ง output จาก คำสั่งหนึ่งไปยังอีกอันหนึ่งได้ ยกตัวอย่างเช่น

```dockerfile
RUN wget -O - https://some.site | wc -l > /number
```

Docker จัดการคำสั่งชุดนี้ด้วย `/bin/sh -c` ซึ่งจะสำเร็จก็ต่อเมื่อคำสั่งสุดท้ายของ pipe นั้นเสร็จสิ้น 

ในตัวอย่างข้างต้นจะเห็นว่าการสร้าง image นี้จะสำเร็จหรือไม่ นั้นขึ้นอยู่กับคำสั่ง `wc -l` โดยไม่คำนึงว่าคำสั่ง `wget` จะสำเร็จหรือไม่ก็ตาม 

ถ้าคุณต้องการให้ทุก error ที่เกิดขึ้นใน pipe หยุดการสร้าง image โดยทันที ให้เพิ่มคำสั่ง `set -o pipefail &&` ไปที่ข้างหน้าของ pipe 

ยกตัวอย่างเช่น

```dockerfile
RUN set -o pipefail && wget -O - https://some.site | wc -l > /number
```
> ไม่ใช่ทุก shells รองรับคำสั่งตัวเลือก `-o pipefail` (`dash` shell บน
> Debian-based images) 
> ในกรณีนั้น ให้ใช้ ชุดคำสั่ง `RUN` ในรูปแบบ ของ _exec_ เพื่อทำการเลือก shell ที่รองรับคำสั่งตัวเลือก `pipefail` ก่อน 
> 
> ยกตัวอย่างเช่น 
>
> ```dockerfile
> RUN ["/bin/bash", "-c", "set -o pipefail && wget -O - https://some.site | wc -l > /number"]
> ```

### CMD

[Dockerfile reference for the CMD instruction](../../engine/reference/builder.md#cmd)

The `CMD` instruction should be used to run the software contained in your
image, along with any arguments. `CMD` should almost always be used in the form
of `CMD ["executable", "param1", "param2"…]`. Thus, if the image is for a
service, such as Apache and Rails, you would run something like `CMD
["apache2","-DFOREGROUND"]`. Indeed, this form of the instruction is recommended
for any service-based image.

In most other cases, `CMD` should be given an interactive shell, such as bash,
python and perl. For example, `CMD ["perl", "-de0"]`, `CMD ["python"]`, or `CMD
["php", "-a"]`. Using this form means that when you execute something like
`docker run -it python`, you’ll get dropped into a usable shell, ready to go.
`CMD` should rarely be used in the manner of `CMD ["param", "param"]` in
conjunction with [`ENTRYPOINT`](../../engine/reference/builder.md#entrypoint), unless
you and your expected users are already quite familiar with how `ENTRYPOINT`
works.

### EXPOSE

[Dockerfile reference for the EXPOSE instruction](../../engine/reference/builder.md#expose)

The `EXPOSE` instruction indicates the ports on which a container listens
for connections. Consequently, you should use the common, traditional port for
your application. For example, an image containing the Apache web server would
use `EXPOSE 80`, while an image containing MongoDB would use `EXPOSE 27017` and
so on.

For external access, your users can execute `docker run` with a flag indicating
how to map the specified port to the port of their choice.
For container linking, Docker provides environment variables for the path from
the recipient container back to the source (ie, `MYSQL_PORT_3306_TCP`).

### ENV

[Dockerfile reference for the ENV instruction](../../engine/reference/builder.md#env)

To make new software easier to run, you can use `ENV` to update the
`PATH` environment variable for the software your container installs. For
example, `ENV PATH /usr/local/nginx/bin:$PATH` ensures that `CMD ["nginx"]`
just works.

The `ENV` instruction is also useful for providing required environment
variables specific to services you wish to containerize, such as Postgres’s
`PGDATA`.

Lastly, `ENV` can also be used to set commonly used version numbers so that
version bumps are easier to maintain, as seen in the following example:

```dockerfile
ENV PG_MAJOR 9.3
ENV PG_VERSION 9.3.4
RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
```

Similar to having constant variables in a program (as opposed to hard-coding
values), this approach lets you change a single `ENV` instruction to
auto-magically bump the version of the software in your container.

Each `ENV` line creates a new intermediate layer, just like `RUN` commands. This
means that even if you unset the environment variable in a future layer, it
still persists in this layer and its value can't be dumped. You can test this by
creating a Dockerfile like the following, and then building it.

```dockerfile
FROM alpine
ENV ADMIN_USER="mark"
RUN echo $ADMIN_USER > ./mark
RUN unset ADMIN_USER
```

```bash
$ docker run --rm test sh -c 'echo $ADMIN_USER'

mark
```

To prevent this, and really unset the environment variable, use a `RUN` command
with shell commands, to set, use, and unset the variable all in a single layer.
You can separate your commands with `;` or `&&`. If you use the second method,
and one of the commands fails, the `docker build` also fails. This is usually a
good idea. Using `\` as a line continuation character for Linux Dockerfiles
improves readability. You could also put all of the commands into a shell script
and have the `RUN` command just run that shell script.

```dockerfile
FROM alpine
RUN export ADMIN_USER="mark" \
    && echo $ADMIN_USER > ./mark \
    && unset ADMIN_USER
CMD sh
```

```bash
$ docker run --rm test sh -c 'echo $ADMIN_USER'

```


### ADD or COPY

- [Dockerfile reference for the ADD instruction](../../engine/reference/builder.md#add)
- [Dockerfile reference for the COPY instruction](../../engine/reference/builder.md#copy)

Although `ADD` and `COPY` are functionally similar, generally speaking, `COPY`
is preferred. That’s because it’s more transparent than `ADD`. `COPY` only
supports the basic copying of local files into the container, while `ADD` has
some features (like local-only tar extraction and remote URL support) that are
not immediately obvious. Consequently, the best use for `ADD` is local tar file
auto-extraction into the image, as in `ADD rootfs.tar.xz /`.

If you have multiple `Dockerfile` steps that use different files from your
context, `COPY` them individually, rather than all at once. This ensures that
each step's build cache is only invalidated (forcing the step to be re-run) if
the specifically required files change.

For example:

```dockerfile
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```

Results in fewer cache invalidations for the `RUN` step, than if you put the
`COPY . /tmp/` before it.

Because image size matters, using `ADD` to fetch packages from remote URLs is
strongly discouraged; you should use `curl` or `wget` instead. That way you can
delete the files you no longer need after they've been extracted and you don't
have to add another layer in your image. For example, you should avoid doing
things like:

```dockerfile
ADD http://example.com/big.tar.xz /usr/src/things/
RUN tar -xJf /usr/src/things/big.tar.xz -C /usr/src/things
RUN make -C /usr/src/things all
```

And instead, do something like:

```dockerfile
RUN mkdir -p /usr/src/things \
    && curl -SL http://example.com/big.tar.xz \
    | tar -xJC /usr/src/things \
    && make -C /usr/src/things all
```

For other items (files, directories) that do not require `ADD`’s tar
auto-extraction capability, you should always use `COPY`.

### ENTRYPOINT

[Dockerfile reference for the ENTRYPOINT instruction](../../engine/reference/builder.md#entrypoint)

The best use for `ENTRYPOINT` is to set the image's main command, allowing that
image to be run as though it was that command (and then use `CMD` as the
default flags).

Let's start with an example of an image for the command line tool `s3cmd`:

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

Now the image can be run like this to show the command's help:

```bash
$ docker run s3cmd
```

Or using the right parameters to execute a command:

```bash
$ docker run s3cmd ls s3://mybucket
```

This is useful because the image name can double as a reference to the binary as
shown in the command above.

The `ENTRYPOINT` instruction can also be used in combination with a helper
script, allowing it to function in a similar way to the command above, even
when starting the tool may require more than one step.

For example, the [Postgres Official Image](https://hub.docker.com/_/postgres/)
uses the following script as its `ENTRYPOINT`:

```bash
#!/bin/bash
set -e

if [ "$1" = 'postgres' ]; then
    chown -R postgres "$PGDATA"

    if [ -z "$(ls -A "$PGDATA")" ]; then
        gosu postgres initdb
    fi

    exec gosu postgres "$@"
fi

exec "$@"
```

> Configure app as PID 1
>
> This script uses [the `exec` Bash command](http://wiki.bash-hackers.org/commands/builtin/exec)
> so that the final running application becomes the container's PID 1. This
> allows the application to receive any Unix signals sent to the container.
> For more, see the [`ENTRYPOINT` reference](../../engine/reference/builder.md#entrypoint).

The helper script is copied into the container and run via `ENTRYPOINT` on
container start:

```dockerfile
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```

This script allows the user to interact with Postgres in several ways.

It can simply start Postgres:

```bash
$ docker run postgres
```

Or, it can be used to run Postgres and pass parameters to the server:

```bash
$ docker run postgres postgres --help
```

Lastly, it could also be used to start a totally different tool, such as Bash:

```bash
$ docker run --rm -it postgres bash
```

### VOLUME

[Dockerfile reference for the VOLUME instruction](../../engine/reference/builder.md#volume)

The `VOLUME` instruction should be used to expose any database storage area,
configuration storage, or files/folders created by your docker container. You
are strongly encouraged to use `VOLUME` for any mutable and/or user-serviceable
parts of your image.

### USER

[Dockerfile reference for the USER instruction](../../engine/reference/builder.md#user)

If a service can run without privileges, use `USER` to change to a non-root
user. Start by creating the user and group in the `Dockerfile` with something
like `RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres`.

> Consider an explicit UID/GID
>
> Users and groups in an image are assigned a non-deterministic UID/GID in that
> the "next" UID/GID is assigned regardless of image rebuilds. So, if it’s
> critical, you should assign an explicit UID/GID.

> Due to an [unresolved bug](https://github.com/golang/go/issues/13548) in the
> Go archive/tar package's handling of sparse files, attempting to create a user
> with a significantly large UID inside a Docker container can lead to disk
> exhaustion because `/var/log/faillog` in the container layer is filled with
> NULL (\0) characters. A workaround is to pass the `--no-log-init` flag to
> useradd. The Debian/Ubuntu `adduser` wrapper does not support this flag.

Avoid installing or using `sudo` as it has unpredictable TTY and
signal-forwarding behavior that can cause problems. If you absolutely need
functionality similar to `sudo`, such as initializing the daemon as `root` but
running it as non-`root`, consider using [“gosu”](https://github.com/tianon/gosu).

Lastly, to reduce layers and complexity, avoid switching `USER` back and forth
frequently.

### WORKDIR

[Dockerfile reference for the WORKDIR instruction](../../engine/reference/builder.md#workdir)

For clarity and reliability, you should always use absolute paths for your
`WORKDIR`. Also, you should use `WORKDIR` instead of  proliferating instructions
like `RUN cd … && do-something`, which are hard to read, troubleshoot, and
maintain.

### ONBUILD

[Dockerfile reference for the ONBUILD instruction](../../engine/reference/builder.md#onbuild)

An `ONBUILD` command executes after the current `Dockerfile` build completes.
`ONBUILD` executes in any child image derived `FROM` the current image.  Think
of the `ONBUILD` command as an instruction the parent `Dockerfile` gives
to the child `Dockerfile`.

A Docker build executes `ONBUILD` commands before any command in a child
`Dockerfile`.

`ONBUILD` is useful for images that are going to be built `FROM` a given
image. For example, you would use `ONBUILD` for a language stack image that
builds arbitrary user software written in that language within the
`Dockerfile`, as you can see in [Ruby’s `ONBUILD` variants](https://github.com/docker-library/ruby/blob/c43fef8a60cea31eb9e7d960a076d633cb62ba8d/2.4/jessie/onbuild/Dockerfile).

Images built with `ONBUILD` should get a separate tag, for example:
`ruby:1.9-onbuild` or `ruby:2.0-onbuild`.

Be careful when putting `ADD` or `COPY` in `ONBUILD`. The "onbuild" image
fails catastrophically if the new build's context is missing the resource being
added. Adding a separate tag, as recommended above, helps mitigate this by
allowing the `Dockerfile` author to make a choice.

## Examples for Official Images

These Official Images have exemplary `Dockerfile`s:

* [Go](https://hub.docker.com/_/golang/)
* [Perl](https://hub.docker.com/_/perl/)
* [Hy](https://hub.docker.com/_/hylang/)
* [Ruby](https://hub.docker.com/_/ruby/)

## Additional resources:

* [Dockerfile Reference](../../engine/reference/builder.md)
* [More about Base Images](baseimages.md)
* [More about Automated Builds](../../docker-hub/builds/index.md)
* [Guidelines for Creating Official Images](../../docker-hub/official_images.md)
