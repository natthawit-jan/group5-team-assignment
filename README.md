
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

คุณสามารถสร้าง image โดยใช้ pipe นำเข้าไฟล์ที่อยู่ใน _local_ หรือ _remote_ ได้ด้วย `stdin` 
วิธีการนี้เป็นประโยชน์มากถ้าคุณต้องการสร้าง image โดยที่ไม่ต้องการบันทึก `Dockerfile` ไว้บนเครื่อง หรือในกรณีที่ `Dockerfile` ถูกสร้างขึ้นมาโดยอัตโนมัติและไม่ต้องการให้ถูกเก็บไว้หลังจากถูกใช้สร้าง image เสร็จ

> ตัวอย่างนี้ใข้เนื้อหาจาก [here documents](http://tldp.org/LDP/abs/html/here-docs.html)
> เพื่อความรวดเร็วในการนำเสนอ แต่คุณสามารถใข้วิธีใดก็ได้ที่สามารถส่ง `Dockerfile` ผ่าน `stdin` 
> เพื่อใช้ฟังค์ชันนี้
> 
> ยกตัวอย่างคำสั่งที่คล้ายกัน 
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
> วิธีการอื่นนอกจากนี้สามารถทำได้ตามความต้องการของคุณได้เลย


#### Build an image using a Dockerfile from stdin, without sending build context

คุณสามารถสร้าง image ด้วย `Dockerfile` ผ่าน `stdin` โดยไม่ต้อง
ใส่ไฟล์อะไรเลยใน build context ด้วยการใส่ (`-`) ตรงตำแหน่งที่ต้องใส่ `PATH` เพื่อสั่งให้ Docker อ่าน  build context (ที่มีแค่ `Dockerfile`) ผ่าน `stdin` แทนที่จะเป็น Directory

```bash
docker build [OPTIONS] -
```

ตัวอย่างการสร้าง image โดยใช้  `Dockerfile` ที่ถูกส่งจาก 
`stdin` จะเห็นได้ว่าไม่มีไฟล์ใดถูกส่งไปเป็น build context เลย

```bash
docker build -t myimage:latest -<<EOF
FROM busybox
RUN echo "hello world"
EOF
```

การไม่ส่ง build context เป็นผลดีในบางกรณี เช่นเมื่อคุณไม่ต้องการ
ให้ไฟล์ใดถูก copy ไปที่ image หรือเมื่อคุณต้องการลดเวลาการสร้าง image เนื่องจากไม่มีไฟล์ใดๆ ถูกส่งไปที่ Docker เลย

ถ้าคุณสนใจในเรื่องการลดเวลาที่ใช้การสร้าง image ด้วยการจัดการไฟล์ ศึกษาต่อได้ที่ [exclude with .dockerignore](#exclude-with-dockerignore).

> **ข้อควรระวัง**: ความพยายามที่จะสร้าง image โดยใช้ชุดคำสั่ง `COPY` หรือ `ADD` จะไม่สำเร็จ
> ถ้าใช้ syntax ต่อไปนี้ 
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

เราจะใช้ syntax นี้เพื่อสร้าง image โดยใช้ไฟล์บนระบบ local แต่ใช้ `Dockerfile`จาก `stdin` โดยการใช้เครื่องหมาย option `-f` (หรือ `--file`) เพื่อระบุ `Dockerfile` ที่จะใช้ และใช้เครื่องหมายขีด (`-` หรือ Hyphen) เพื่อบอกชื่อไฟล์ให้ Docker ไปอ่าน `Dockerfile` จาก `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

ตัวอย่างข้างล่างนี้ใช้ directory ปัจจุบัน (`.`) เป็น build context และสร้าง image โดยการใช้ `Dockerfile` ผ่าน `stdin` โดยการใช้[เอกสารชุดนี้](http://tldp.org/LDP/abs/html/here-docs.html)

```bash
# สร้าง directory ที่จะทำงาน
mkdir example
cd example

# สร้างไฟล์ตัวอย่าง
touch somefile.txt

# สร้าง image โดยใช้ directory ปัจจุบันเป็น context และ Dockerfile ที่ผ่าน stdin
docker build -t myimage:latest -f- . <<EOF
FROM busybox
COPY somefile.txt .
RUN cat /somefile.txt
EOF
```

#### Build from a remote build context, using a Dockerfile from stdin

เราจะใช้ syntax นี้เพื่อสร้าง image โดยใช้ไฟล์จากรีโมทของ `git` repository และใช้ `Dockerfile`จาก `stdin` โดยการใช้เครื่องหมาย option `-f` (หรือ `--file`) เพื่อระบุ `Dockerfile` ที่จะใช้ และใช้เครื่องหมายขีด (`-` หรือ Hyphen) เพื่อบอกชื่อไฟล์ให้ Docker ไปอ่าน `Dockerfile` จาก `stdin`:

```bash
docker build [OPTIONS] -f- PATH
```

Syntax จะมีประโยชน์มากในเวลาที่เราต้องการที่จะสร้าง image จาก repository ที่ไม่มี `Dockerfile` อยู่ในนั้น หรือสร้าง custom `Dockerfile` โดยที่ไม่เก็บ fork ของตัวเองใน repository

ตัวอย่างข้างล่างเป็นการสร้าง image โดยใช้ `Dockerfile` จาก `stdin` และเพิ่มไฟล์ `hello.c` จาก ["hello-world" Git repository บน GitHub](https://github.com/docker-library/hello-world)

```bash
docker build -t myimage:latest -f- https://github.com/docker-library/hello-world.git <<EOF
FROM busybox
COPY hello.c .
EOF
```

> **Under the hood**
>
> เมื่อสร้าง image โดยใช้ remote Git repository เป็น build context แล้ว
> Docker จะใช้คำสั่ง `git clone` ของ repository บนเครื่อง local และส่ง
> ไฟล์เหล่านั้นเป็น build context ไปที่ daemon โดยที่ feature นี้ต้องใช้ `git` 
> เพื่อที่จะถูกติดตั้งที่ host ที่เราจะสามารถ run คำสั่ง `docker build` ได้

### Exclude with .dockerignore

เราจะใช้ไฟล์ `.dockerignore` เพื่อที่จะไม่ต้อง build ไฟล์ที่ไม่เกี่ยวข้องรวมไปด้วย (ไม่ต้องจัดโครงสร้างของ source repository ใหม่) ไฟล์นี้รองรับ exclusion patterns คล้ายกับไฟล์ `.gitignore` ส่วนข้อมูลในการสร้างเพิ่มเติม สามารถดูได้ที่ [.dockerignore file](../../engine/reference/builder.md#dockerignore-file)


### Use multi-stage builds

[Multi-stage builds](multistage-build.md) ช่วยให้ลดขนาดของ final image อย่างมากโดยที่ไม่ต้องไปยุ่งกับการลดจำนวนของ intermediate layers และไฟล์ 

เพราะว่า image ถูกสร้างระหว่างขั้นตอนสุดท้ายของกระบวนการ build เราสามารถลด image layers โดย [leveraging build cache](#leverage-build-cache)

ตัวอย่างเช่นถ้า build ของเรามีหลาย layers เราสามารถบอกให้พวกมันจากเปลี่ยนแปลงไม่บ่อย (เพื่อที่จะมั่นใจได้ว่า build cache สามารถนำกลับมาใช้ใหม่ได้) เป็นให้เปลี่ยนแปลงบ่อยขึ้นได้

* ติดตั้งเครื่องมือที่ต้องการเพื่อ build application ของคุณ

* ติดตั้ง หรือ อัพเดท library dependencies

* สร้าง application

Dockerfile ของ application Go จะเป็นแบบดังนี้:

```dockerfile
FROM golang:1.11-alpine AS build

# ติดตั้ง tools ที่ต้องใช้สำหรับ project
# รัน `docker build --no-cache .` เพื่ออัพเดท dependencies
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep

# List project dependencies ด้วย Gopkg.toml และ Gopkg.lock
# Layers เหล่านี้จะถูก rebuild ก็ต่อเมื่อไฟล์ Gopkg ถูกอัพเดทเท่านั้น
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# ติดตั้ง library dependencies
RUN dep ensure -vendor-only

# คัดลอก project ทั้งหมดและ build ใหม่
# Layer นี้จะถูก rebuild ก็ต่อเมื่อไฟล์มีการเปลี่ยนแปลงใน project directory
COPY . /go/src/project/
RUN go build -o /bin/project

# ผลที่ออกมาในรูปของ single layer image
FROM scratch
COPY --from=build /bin/project /bin/project
ENTRYPOINT ["/bin/project"]
CMD ["--help"]
```


### Don't install unnecessary packages

ควรหลีกเลี่ยงที่จะติดตั้ง packages อื่นๆที่ไม่จำเป็น เพื่อที่จะได้ช่วยลดความซับซ้อน ลดส่วนของซอฟแวร์ที่ไม่จำเป็น ลดขนาดของไฟล์ และลดระยะเวลาในการ build ไม่ใช่เพียงเพราะคุณเห็นเป็น packages ที่อยากจะเก็บไว้ เช่น คุณไม่จำเป็นต้องใช้ text editor ใน database image.

### Decouple application

Container ควรมีเพียง concern เดียว การแยก applications 
เป็นหลาย container ทำให้ง่ายต่อการปรับขนาดในแนวนอนและการนำ container มาใช้ใหม่
ตัวอย่างเช่น web application stack ประกอบไปด้วย 3 container 
แยกกันแต่ละอัน มี image เฉพาะเพื่อจัดการ web application ฐานข้อมูลและ cache หน่วยความจำที่แยกกัน


การจำกัดแต่ละ container ในหนึ่งกระบวนการเป็นกฎง่ายๆเป็นกระบวนการเริ่มต้น
บางโปรแกรมอาจเป็นกระบวนการเพิ่มเติมตามข้อตกลง ตัวอย่างเช่น Celery เป็น 
กระบวนการที่ทำงานหลายคน และ Apache สามารถสร้างหนึ่งกระบวนการต่อคำขอ


เพื่อให้ container ที่ดีที่สุดที่เป็นไปได้ หากขึ้นอยู่กับแต่ละ container 
สามารถใช้ Docker container networks เพื่อให้แน่ใจว่าคอนเทนเนอร์เหล่านี้สามารถสื่อสารได้

### Minimize the number of layers

ในเวอร์ชั่นเก่าของ Docker การลดตัวเลขของลำดับชั้น (layers) ใน images เป็นสิ่งสำคัญที่จะยืนยันว่า images ของคุณ มีประสิทธิภาพ
ซึ่ง Features ดังที่จะกล่าวถึง ถูกเพิ่มเข้าไปเพื่อลดข้อจำกัดดังต่อไปนี้

- คำสั่ง `RUN`, `COPY`, `ADD` ใช้เพื่อสร้างลำดับชั้น (Layers) เท่านั้น. คำสั่งอื่นๆ ใช้สร้างตัวกลาง images ชั่วคราว
และห้ามเพิ่มขนาดของการสร้าง (build)

- ถ้าหากเป็นไปได้ ควรใช้การ build ในหลายๆขั้นตอน ([multi-stage builds](multistage-build.md)) และคัดลอก artifacts ที่คุณต้องการลงใน image 
ขั้นสุดท้าย ซึ่งจะทำให้คุณได้รวบรวมเครื่องมือ (tools) และข้อมูลการดีบัค (debug information) ในช่วงกลางของขั้นตอนการ build 
โดยที่ image ขั้นสุดท้าย จะไม่มีการเพิ่มขนาดให้ใหญ่ขึ้นกว่าเดิม


### Sort multi-line arguments

เพื่อที่จะทำให้ใช้งานได้ง่ายขึ้น ถ้าเป็นไปได้ควรเรียง argument แบบหลายๆบรรทัด (multi-line) ตามลำดับตัวเลขและอักขระ
ซึ่งจะช่วยหลีกเลี่ยงการซ้อนทับกันของ packages และทำให้ง่ายต่อการอัพเดต  list มากกว่า
และยังทำให้ PRs ง่ายต่อการอ่านและ review และการใส่ช่องว่าง (space) ก่อนเครื่องหมาย backslash (`\`)
ก็สามารถช่วยได้เช่นกัน

ตัวอย่าง [`buildpack-deps` image](https://github.com/docker-library/buildpack-deps):

```dockerfile
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion
```

### Leverage build cache

ตอนที่สร้าง image, Docker จะทำตามขั้นตอน instructions ตามในไฟล์ `Dockerfile` โดยไล่มาเป็นลำดับ และในแต่ละ instruction ที่ถูกอ่าน Docker จะไปหา image ที่มีอยู่แล้วก่อนใน cache ที่สามารถนำกลับมาใช้ใหม่ได้ (reuse) ก่อนที่จะไปสร้าง image ใหม่

ถ้าเราไม่ต้องการใช้ cache เราสามารถใช้ option `--no-cache=true` ที่คำสั่ง `docker build` ได้ แต่ว่าถ้าเราให้ Docker ใช้ cache ของตัวมันเอง เราจำเป็นต้องเข้าใจว่ามันจะหา image ที่ match ได้หรือไม่ได้ ต่อจากนี้คือกฏเบื้องต้นที่ Docker จะปฏิบัติตามเป็นดังนี้:

- เริ่มจาก parent image ที่มีอยู่แล้วใน cache, instruction ถัดไปจะถูกเปรียบเทียบกับ child images
  ทั้งหมดที่ถูกแปลงมาจาก base image เพื่อดูว่ามีตัวไหนไหมที่ถูก build โดยใช้ instruction เดียวกัน ถ้าไม่
  cache จะถูกทำให้เป็นโมฆะ       

- โดยทั่วไป การเปรียบเทียบ  instruction ใน `Dockerfile` สักตัวหนึ่งใน child image ก็เพียงพอแล้ว 
  แต่ว่าบาง instruction ต้องการการรวจสอบและการขยายความที่มากขึ้น

- สำหรับ instruction `ADD` และ `COPY` นั้น content ของไฟล์ใน image ถูก examine และมีการ checksum 
  ในทุกๆไฟล์ โดยที่ last-modified และ last-accessed times ของไฟล์ไม่ได้ถูกคิดไว้ใน checksum ด้วย 
  และในระหว่างการค้นหา cache นั้น checksum จะถูกเทียบกับ image ที่มีอยู่ก่อนหน้า 
  ถ้าสมมติว่ามีไฟล์ที่ถูกเปลี่ยนแปลงในนั้น เช่น contents และ metadata, cache จะเป็นโมฆะทันที

- นอกจากคำสั่ง `ADD` และ `COPY` แล้ว cache checking ก็ไม่ได้ไปดูไฟล์ใน container เพื่อที่จะทำการ cache
  match เลย ยกตัวอย่างเช่น เมื่อรันคำสั่ง `RUN apt-get -y update` ไฟล์ที่ถูกอัพเดทใน container 
  จะไม่ถูกนำมาชี้ว่ามี cache hit อยู่ไหม ในกรณีนี้จะมีแค่ command string เท่านั้นที่ถูกใช้ในการหา match

เมื่อใดที่ cache เป็นโมฆะแล้ว คำสั่งที่ตามมาใน `Dockerfile` ทั้งหมดจะถูกสร้างขึ้นมาใหม่ และ cache จะไม่ถูกใช้

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

[การอ้างอิง Dockerfile สำหรับคำสั่ง CMD](../../engine/reference/builder.md#cmd)

ควรใช้คำสั่ง CMD เพื่อเรียกใช้ซอฟต์แวร์ที่มีอยู่ในimageของคุณ ควรใช้ CMD ในรูปแบบของ CMD ["executable", 
"param1", "param2"…] เสมอ ถ้า image สำหรับบริการต่างๆ เช่น Apache and Rails คุณจะต้อง run บางอย่าง
เหมือน CMD ["apache2","-DFOREGROUND"] จริงๆแล้วรูปแบบของคำสั่งนี้แนะนำสำหรับ image ที่ใช้ service ต่างๆ

ในกรณีอื่นๆ ส่วนใหญ่ CMD ควรจะเป็น shell แบบโต้ตอบ เช่น bash, python และ perl ตัวอย่างเช่น CMD ["perl", 
"-de0"], CMD ["python"], or CMD ["php", "-a"] การใช้แบบฟอร์มนี้หมายความว่า เมื่อคุณดำเนินการบางอย่าง 
เช่น docker run -it python คุณจะอยู่ใน shell ที่พร้อมใช้งาน ไม่ควรใช้ CMD ในลักษณะของ CMD ["param", "param"] 
ร่วมกับ ENTRYPOINT เว้นแต่ว่าคุณและ users คุ้นเคยกับวิธีการทำงานของ ENTRYPOINT

### EXPOSE

[การอ้างอิง Dockerfile สำหรับคำสั่ง EXPOSE](../../engine/reference/builder.md#expose)

คำสั่งEXPOSE ชี้ว่า PORT ที่ Container รับฟังการเชื่อมต่อ ดังนั้นคุณควรใช้พอร์ตทั่วไปสำหรับแอปพลิเคชันของคุณ 
ตัวอย่างเช่นรูป image ที่มี Apache web server จะใช้ EXPOSE 80 ในขณะที่ image ที่มี MongoDB 
จะใช้ EXPOSE 27017 เป็นต้น

สำหรับการเข้าถึงจากภายนอก users สามารถเรียกใช้ docker run with a flag ที่มีการระบุวิธีแมปพอร์ตที่ระบุกับพอร์ตที่
เลือกไว้ สำหรับการเชื่อมต่อ Container,  Docker จัดเตรียมตัวแปรสภาพแวดล้อมสำหรับpathจากContainer กลับไปที่
ต้นทาง (ie, MYSQL_PORT_3306_TCP).

### ENV

[อ้างอิง Dockerfile สำหรับ ENV ](../../engine/reference/builder.md#env)

การทำให้ซอฟแวร์ใหม่ๆง่ายที่จะประมวลผล คุณสามารถใช้  `ENV` เพื่ออัพเดต `PATH` environment ของตัวแปรสำหรับซอฟแวร์ที่ container ของคุณได้ติดตั้ง ยกตัวอย่าง เช่น `ENV PATH /usr/local/nginx/bin:$PATH` ยืนยันได้ว่า `CMD ["nginx"]` นั้น ใช้งานได้

คำสั่ง `ENV`  ก็ยังเป็นประโยชน์สำหรับการจัดหา environment ของตัวแปรที่ต้องการ ซึ่งจะให้บริการในสิ่งที่คุณต้องการ containerize เช่น Postgres’s`PGDATA`

ท้ายที่สุดแล้ว `ENV` ยังสามารถใช้เพื่อตั้งค่าหมายเลขของเวอร์ชั่นที่ต้องการใช้ ดังนั้นเมื่อเกิด version bumps ก็จะสามารถปรับปรุงได้ง่ายขึ้น ดังเช่นตัวอย่างต่อไปนี้

```dockerfile
ENV PG_MAJOR 9.3
ENV PG_VERSION 9.3.4
RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
```
เปรียบเสมือนกับที่ในโปรแกรมมีค่าตัวแปรคงที่ (ซึ่งจะตรงข้ามกับ hard-coding values)
วิธีการนี้จะทำให้คุณเปลี่ยนแค่ `ENV` เดียว เพื่อที่จะได้ bump เวอร์ชั่นของซอฟแวร์ใน container  ของคุณโดยอัตโนมัติ

ในแต่ละบรรทัดของ `ENV` จะสร้าง layer ใหม่ ซึ่งจะเหมือนกับคำสั่ง `RUN`
โดยแสดงให้เห็นว่า ถึงแม้ว่าคุณจะยกเลิกการตั้งค่า environment ของตัวแปรใน layer ที่จะเกิดขึ้นข้างหน้า
แต่มันก็ยังคงมีอยู่ใน layer นี้ และค่าของมันก็จะไม่สามารถ dump ได้
โดยคุณสามารถทดสอบด้วยการสร้าง Dockerfile เหมือนดังต่อไปนี้ และหลังจากนั้นจึงทำการ build 

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

เพื่อป้องกันเหตุการณ์นี้ และเมื่อไม่ได้ยกเลิกการตั้งค่า environment ของตัวแปร
ใช้คำสั่ง `RUN` ด้วย shell commands เพื่อทำการ ตั้งค่า ใช้ และยกเลิกการตั้งค่าตัวแปรทั้งหมดใน layer เดี่ยว
คุณสามารถแยกคำสั่งด้วยเครื่องหมาย `;` หรือ `&&` 
ถ้าหากคุณใช้วิธีที่ 2 และหนึ่งในคำสั่งนั้นล้มเหลว ก็จะทำให้ `docker build` ล้มเหลวไปด้วย ซึ่งนี่ก็เป็นอีกหนึ่งไอเดียที่ดี
การใช้เครื่องหมาย `\` ในการเว้นบรรทัดตัวอักษรสำหรับ Dockerfiles ใน Linux สามารถทำให้การอ่านมีประสิทธิภาพยิ่งขึ้น
คุณสามารถใช้ทุกคำสั่งใน shell script และกด `RUN` เพื่อที่จะได้ทำการ `RUN` shell script นั้น

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

ADD และ COPY มีฟังก์ชั่นคล้ายกัน โดยทั่วไปแล้วเราจะใช้ COPY
COPY จะรองรับการคัดลอก Local filesไปยัง Container ในขณะที่ ADD สามารถแตกไฟล์ .tar 
และสามารถดาวโหลดไฟล์จาก url ภายนอกได้ ดังนั้น การใช้ ADD ที่ดีที่สุด คือการดึงไฟล์ .tar เช่นเดียวกับใน ADD rootfs.tar.xz 
หากมี dockfile หลายขั้นตอน ที่ใช้ไฟล์แตกต่างกัน คัดลอกเป็นรายบุคคล ทำทุกอย่างในครั้งเดียว
ซึ่งช่วยให้มั่นใจว่าแคชการสร้างของแต่ละขั้นตอนจะ invalidated เท่านั้น 


For example:

```dockerfile
COPY requirements.txt /tmp/
RUN pip install --requirement /tmp/requirements.txt
COPY . /tmp/
```
ผลลัพธ์ใน invalidations แคชน้อยลงสําหรับขั้นตอน RUN, น้อยกว่าถ้าใส่ 
`COPY . /tmp/` before it.

เพราะ image size มีความสำคัญ การใช้ ADD ดึง packages จาก remote URL นั้นไม่รับรอง 
ควรใช้ curl หรือ wget แทน วิธีนี้จะทำให้สามารถลบไฟล์ที่ไม่ต้องการได้
หลังจากที่แตกไฟล์ออกแล้วและไม่จำเป็นต้องเพิ่ม Layer อื่น เช่น

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
สำหรับรายการอื่น ๆ (files, directories) ที่ไม่ต้องการความสามารถในการแยกอัตโนมัติของ ADD ควรใช้ COPY

### ENTRYPOINT

[Dockerfile reference for the ENTRYPOINT instruction](../../engine/reference/builder.md#entrypoint)

การใช้งานที่ดีที่สุดสำหรับ ENTRYPOINT คือ set คำสั่ง image,อนุญาตให้ Image run ผ่านคำสั่ง (และใช้ CMD เป็นค่าเริ่มต้น) 

Let's start with an example of an image for the command line tool `s3cmd`:

```dockerfile
ENTRYPOINT ["s3cmd"]
CMD ["--help"]
```

Image สามารถเรียกใช้เพื่อแสดงวิธีใช้คำสั่ง

```bash
$ docker run s3cmd
```

หรือใช้ parameters ที่เหมาะสม Run คำสั่ง

```bash
$ docker run s3cmd ls s3://mybucket
```

สิ่งนี้มีประโยชน์เพราะ Image name สามารถเพิ่มเป็น double ของการอ้างอิงถึง Binary ดังที่แสดงคำสั่งด้านบน

คำสั่ง ENTRYPOINT สามารถใช้ร่วมกับ helper script, 
อนุญาตให้ฟังก์ชั่นทำงานที่คล้ายกันกับคำสั่งข้างบน 
แม้เมื่อเริ่มต้นเครื่องมืออาจจำเป็นต้องมากกว่าหนึ่งขั้นตอน


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

> กำหนดแอพเป็น PID 1
script นี้ใช้คำสั่ง exec Bash ดังนั้นแอปพลิเคชั่นสุดท้ายที่ทำงานจะกลายเป็น PID ของ container1 สิ่งนี้อนุญาตให้แอปพลิเคชันรับสัญญาณ Unix ใด ๆ ส่งไปยัง container, สำหรับข้อมูลเพิ่มเติมดูการอ้างอิง ENTRYPOINT

Helper script จะถูก copy ลงใน container และเรียกใช้ผ่าน ENTRYPOINT เมื่อเริ่มต้น container

```dockerfile
COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["postgres"]
```

script นี้อนุญาตให้ผู้ใช้โต้ตอบกับ Postgres ได้หลายวิธี

It can simply start Postgres:

```bash
$ docker run postgres
```

หรือสามารถเรียกใช้ Postgres และส่ง parameters ไปยัง server

```bash
$ docker run postgres postgres --help
```

สุดท้ายก็สามารถเริ่มใช้กับเครื่องมือที่แตกต่างเช่น Bash

```bash
$ docker run --rm -it postgres bash
```

### VOLUME

[Dockerfile reference for the VOLUME instruction](../../engine/reference/builder.md#volume)

ควรใช้คำสั่ง VOLUME เพื่อแสดงพื้นที่เก็บข้อมูลฐาน ข้อมูลการจัดเก็บการกำหนดค่าหรือไฟล์ 
โฟลเดอร์ที่สร้างโดย docker container ควรใช้ VOLUME สำหรับพื้นที่ที่ไม่แน่นอนหรือที่ผู้ใช้สามารถแก้ไขได้ส่วนของ image


### USER

[Dockerfile reference for the USER instruction](../../engine/reference/builder.md#user)

ถ้าหาก service สามารถทำงานโดยไม่ต้องใช้สิทธิพิเศษ ใช้ USER เพื่อเปลี่ยนแปลง non-root user 
เริ่มต้นโดยการสร้าง User และกลุ่มใน Dockerfile อย่างเช่น
`RUN groupadd -r postgres && useradd --no-log-init -r -g postgres postgres`.

> Consider an explicit UID/GID
>
> User และกลุ่มใน image ถูกกำหนด UID / GID ใน "next" คือ กำหนดโดยไม่คำนึงถึงการสร้าง image ใหม่
> ดังนั้น เราควรกำหนด UID / GID ที่ชัดเจน


> เนื่องจากมีข้อผิดพลาดที่ไม่ได้รับการแก้ไขในการจัดการเก็บไฟล์ถาวร 
สร้าง User ขนาดใหญ่ที่มี UID ภายใน Container Docker  
ไปยัง disk เนื่องจาก `/var/log/faillog` ใน Layer container 
เต็มไปด้วย NULL (\0) วิธีแก้ปัญหาคือการส่ง `--no-log-init` ไปที่ useradd

หลีกเลี่ยงการติดตั้งหรือใช้ sudo ที่มีลักษณะการทำงานของ TTY และการส่งต่อสัญญาณที่อาจทำให้เกิดปัญหาารถ 
ถ้าต้องการฟังก์ชั่นที่คล้ายกับ sudo เช่นการเริ่มต้น daemon เป็น Root แต่ run เป็น non-root ให้ลองใช้“ gosu”

สุดท้ายเพื่อลด Layer และความซับซ้อนให้หลีกเลี่ยงการสลับ USER 

### WORKDIR

[อ้างอิง Dockerfile สำหรับคำแนะนำของ WORKDIR](../../engine/reference/builder.md#workdir)

เพื่อความชัดเจนและปราศจากข้อบกพร่อง คุณควรใช้ absolute paths สำหรับ `WORKDIR` ของคุณ
และคุณควรใช้ `WORKDIR`แทนการใช้คำสั่งยาวๆ เช่น `RUN cd … && do-something`
ซึ่งจะยากในการอ่าน การแก้ไขปัญหา และการปรับปรุง

### ONBUILD

[การอ้างอิง Dockerfile สำหรับคำสั่ง ONBUILD ](../../engine/reference/builder.md#onbuild)

คำสั่ง `ONBUILD` ทำหลังจากการสร้าง `Dockerfile` ปัจจุบันเสร็จสมบูรณ์.
`ONBUILD` ดำเนินการใน child image ที่ได้รับจาก image ปัจจุบัน .  คิดว่าคำสั่ง `ONBUILD` เป็นคำสั่งที่ parent `Dockerfile` มอบให้  child `Dockerfile`.

Docker build เรียกใช้คำสั่ง `ONBUILD` ก่อนคำสั่งใดๆ ใน child
`Dockerfile`.

`ONBUILD` มีประโยชน์สำหรับ image ที่กำลังจะสร้างจาก image ที่กำหนด ตัวอย่างเช่นคุณจะใช้  `ONBUILD` 
มีประโยชน์สำหรับ image ที่กำลังจะสร้างจาก image ที่กำหนด ตัวอย่างเช่นคุณจะใช้ 
`Dockerfile`, เหมือนที่คุณเห็นในตัวแปร  [Ruby’s `ONBUILD` variants](https://github.com/docker-library/ruby/blob/c43fef8a60cea31eb9e7d960a076d633cb62ba8d/2.4/jessie/onbuild/Dockerfile).

Image ที่สร้างด้วย  `ONBUILD` ควรได้รับtagแยกต่างหากตัวอย่างเช่น:
`ruby:1.9-onbuild` or `ruby:2.0-onbuild`.

ระวังเมื่อใส่ `ADD` หรือ `COPY` ใน `ONBUILD`. "onbuild" image
ล้มเหลวอย่างหายนะถ้าหาก new build’s context ขาด resource ที่เพิ่มเข้ามา .การเพิ่ม tag แยก 
ตามที่แนะนำข้างต้นจะช่วยลดปัญหานี้โดยอนุญาตให้ผู้เขียน `Dockerfile` ทำการสร้างตัวเลือก
