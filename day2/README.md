# 2일차. 스트림 데이터 수집

> 플루언트디를 통해 다양한 수집 예제를 실습합니다 이번 장에서 사용하는 외부 오픈 포트는 22, 80, 5601, 8080, 9880, 50070 입니다

- 범례
  * :green_book: : 기본, :blue_book: : 중급, :closed_book: : 고급
- 목차
  * [1. 파일 수집 기본](#1-파일-수집-기본)
    - [1. 최신버전 업데이트](#1-최신버전-업데이트)
    - [2. 플루언트디를 웹서버처럼 사용하기](#2-플루언트디를-웹서버처럼-사용하기)
    - [3. 수신된 로그를 로컬에 저장하는 예제](#3-수신된-로그를-로컬에-저장하는-예제)
    - [4. 서버에 남는 로그를 지속적으로 모니터링하기](#4-서버에-남는-로그를-지속적으로-모니터링하기)
  * [2. 파일 수집 고급](#2-파일-수집-고급)
    - [5. 데이터 변환 플러그인을 통한 시간 변경 예제](#5-데이터-변환-플러그인을-통한-시간-변경-예제)
    - [6. 컨테이너 환경에서의 로그 전송](#6-컨테이너-환경에서의-로그-전송)
    - [7. 도커 컴포즈를 통한 로그 전송 구성](#7-도커-컴포즈를-통한-로그-전송-구성)
    - [8. 멀티 프로세스를 통한 성능 향상](#8-멀티-프로세스를-통한-성능-향상)
    - [9. 멀티 프로세스를 통해 하나의 위치에 저장](#9-멀티-프로세스를-통해-하나의-위치에-저장)
    - [10. 전송되는 데이터를 분산 저장소에 저장](#10-전송되는-데이터를-분산-저장소에-저장)
  * [3. 연습문제](#3-연습문제)
    * [3-1. 참고 사이트](#3-1-참고-사이트)
    * [3-2. 퀴즈](#3-2-퀴즈)

<br>


## 1. 파일 수집 기본

## 1. 최신버전 업데이트
> 원격 터미널에 접속하여 관련 코드를 최신 버전으로 내려받고, 과거에 실행된 컨테이너가 없는지 확인하고 종료합니다

### 1-1. 최신 소스를 내려 받습니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training
git pull
```
<br>

### 1-2. 현재 기동되어 있는 도커 컨테이너를 확인하고, 종료합니다

#### 1-2-1. 현재 기동된 컨테이너를 확인합니다
```bash
# terminal
docker ps -a
```
<br>

#### 1-2-2. 기동된 컨테이너가 있다면 강제 종료합니다
```bash
# terminal 
docker rm -f `docker ps -aq`
```
> 다시 `docker ps -a` 명령으로 결과가 없다면 모든 컨테이너가 종료되었다고 보시면 됩니다
<br>


### 1-3. 이번 실습은 예제 별로 다른 컨테이너를 사용합니다

> `cd ~/work/data-engineer-advanced-training/day2/`<kbd>ex1</kbd> 와 같이 마지막 경로가 다른 점에 유의 하시기 바랍니다

* 1번 실습의 경로는 <kbd>ex1</kbd>이므로 아래와 같습니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex1
```

[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>



## 2. 플루언트디를 웹서버처럼 사용하기

> 플루언트디 로그 수집기는 **에이전트 방식의 데몬 서버**이기 때문에 일반적인 웹서버 처럼 *http 프로토콜을 통해서도 로그를 전송받을 수 있습*니다

* 가장 일반적인 데이터 포맷인 *JSON 으로 로그를 수신하고, 표준출력(stdout)으로 출력*하는 예제를 실습해봅니다
* 다른 부서 혹은 팀내의 다른 애플리케이션 간의 **로그 전송 프로토콜이 명확하지 않은 경우 가볍게 연동해볼 만한 모델**입니다

![ex1](images/ex1.png)

### 2-1. 도커 컨테이너 기동
```bash
cd ~/work/data-engineer-advanced-training/day2/ex1
docker-compose pull
docker-compose up -d
```
<br>


### 2-2. 도커 및 플루언트디 설정파일 확인

> 기본 설정은 /etc/fluentd/fluent.conf 파일을 바라보는데 예제 환경에서는 `docker-compose.yml` 설정에서 해당 위치에 ex1/fluent.conf 파일을 마운트해 두었기 때문에 컨테이너 환경 내에서 바로 실행을 해도 잘 동작합니다. 별도로 `fluentd -c /etc/fluentd/fluent.conf` 로 실행해도 동일하게 동작합니다

#### 2-2-1 도커 컴포즈 파일 구성 `docker-compose.yml`

```yaml
version: "3"

services:
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    ports:
      - 9880:9880
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
      - ./send_http.sh:/home/root/send_http.sh
    working_dir: /home/root

networks:
  default:
    name: day2_ex1_network
```
> 플루언트디가 9880 포트를 통해 http 웹서버를 기동했기 때문에 컴포즈 파일에서 해당 포트를 노출시킨 점도 확인합니다

#### 2-2-2 플루언트디 파일 구성 `fluent.conf`
```xml
<source>
    @type http
    port 9880
    bind 0.0.0.0
</source>

<match test>
    @type stdout
</match>
```
<br>


### 2-3. 플루언트디 기동 및 확인

#### 2-3-1. 에이전트 기동을 위해 컨테이너로 접속 후, 에이전트를 기동합니다
```bash
# terminal
docker-compose exec fluentd bash
```
```bash
# docker
fluentd
```
* `fluentd -c /etc/fluentd/fluent.conf`로 기동해도 동일하게 동작합니다

<details><summary> :green_book: 1. [기본] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```bash
[Default] Start fluentd agent w/ default configuration
/opt/td-agent/embedded/bin/fluentd -c /etc/fluentd/fluent.conf
2021-07-18 11:03:23 +0000 [info]: parsing config file is succeeded path="/etc/fluentd/fluent.conf"
2021-07-18 11:03:23 +0000 [info]: using configuration file: <ROOT>
  <source>
    @type http
    port 9880
    bind "0.0.0.0"
  </source>
  <match test>
    @type stdout
  </match>
</ROOT>
2021-07-18 11:03:23 +0000 [info]: starting fluentd-1.11.5 pid=22 ruby="2.4.10"
2021-07-18 11:03:23 +0000 [info]: spawn command to main:  cmdline=["/opt/td-agent/embedded/bin/ruby", "-Eascii-8bit:ascii-8bit", "/opt/td-agent/embedded/bin/fluentd", "-c", "/etc/fluentd/fluent.conf", "--under-supervisor"]
2021-07-18 11:03:24 +0000 [info]: adding match pattern="test" type="stdout"
2021-07-18 11:03:24 +0000 [info]: adding source type="http"
2021-07-18 11:03:24 +0000 [info]: #0 starting fluentd worker pid=31 ppid=22 worker=0
2021-07-18 11:03:24 +0000 [info]: #0 fluentd worker is now running worker=0
```

</details>
<br>

#### 2-3-2. 별도의 터미널에서 cURL 명령으로 확인합니다

* 웹 브라우저를 통해 POST 전달이 안되기 때문에 별도 터미널로 접속합니다
  - 클라우드 터미널에 curl 설치가 되어있지 않을 수도 있으므로 도커 컨테이너에 접속합니다
```bash
# terminal
docker-compose exec fluentd bash
```
```bash
# docker
curl -i -X POST -d 'json={"action":"login","user":2}' http://localhost:9880/test
# HTTP/1.1 200 OK
# Content-Type: text/plain
# Connection: Keep-Alive
# Content-Length: 0
```
> 사전에 배포된 `send_http.sh` 를 실행해도 동일한 결과를 얻습니다

<details><summary> :green_book: 2. [기본] 출력 결과 확인</summary>

> 에이전트가 기동된 컨테이너의 화면에는 아래와 같이 수신된 로그를 출력하면 성공입니다

```text
2021-07-18 11:12:47.866146971 +0000 test: {"action":"login","user":2}
2021-07-18 11:13:38.334081170 +0000 test: {"action":"login","user":2}
```

</details>
<br>


#### 2-3-3. 실습이 끝났으므로 플루언트디 에이전트를 컨테이너 화면에서 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 종료합니다

![stop](images/stop.png)

#### 2-3-4. 1번 예제 실습이 모두 종료되었으므로 <kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 혹은 <kbd>exit</kbd> 명령으로 컨테이너를 종료합니다

> 컨테이너가 종료된 터미널의 프롬프트(도커의 경우 root@2a50c30293829 형식)를 통해 확인할 수 있습니다


[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>



## 3. 수신된 로그를 로컬에 저장하는 예제

> 로그를 전송해 주는 서비스 혹은 애플리케이션을 별도로 기동하는 것도 귀찮은 일이기 때문에, 플루언트디에서 제공하는 로그를 자동으로 발생시키는 더미 로그 플러그인을 이용하여 로그를 발생시키고, 발생된 로그를 로컬 저장소에 저장하는 예제를 실습합니다

* 플루언트디 더미 에이전트를 이용하여 임의의 로그를 자동으로 발생시킴
* 발생된 로그(이벤트)를 로컬 저장소에 임의의 경로에 저장합니다

![ex2](images/ex2.png)
<br>


### 3-1. 이전 컨테이너 종료 및 컨테이너 기동

> 이전 실습에서 기동된 컨테이너를 종료 후, 기동합니다.

```bash
cd ~/work/data-engineer-advanced-training/day2/ex1
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex2
docker-compose pull
docker-compose up -d
```
<br>

#### 3-1-1 도커 컴포즈 파일 구성 `docker-compose.yml`

```yaml
version: "3"

services:
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    ports:
      - 9880:9880
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
      - ./target:/fluentd/target
    working_dir: /home/root

networks:
  default:
    name: day2_ex2_network
```
> 저장되는 로그를 호스트 장비의 볼륨에 마운트하여 컨테이너가 종료되는 경우에도 유지될 수 있도록 구성합니다

<br>


#### 3-1-2 플루언트디 파일 구성 `fluent.conf`
```xml
<source>
    @type dummy
    tag lgde.info
    size 5
    rate 1
    auto_increment_key seq
    dummy {"info":"hello-world"}
</source>

<source>
    @type dummy
    tag lgde.debug
    size 3
    rate 1
    dummy {"debug":"hello-world"}
</source>

<filter lgde.info>
    @type record_transformer
    <record>
        table_name ${tag_parts[0]}
    </record>
</filter>

<match lgde.info>
    @type file
    path_suffix .log
    path /fluentd/target/${table_name}/%Y%m%d/part-%Y%m%d.%H%M
    <buffer time,table_name>
        timekey 1m
        timekey_wait 10s
        timekey_use_utc false
        timekey_zone +0900
    </buffer>
</match>

<match lgde.debug>
    @type stdout
</match>
```
> 더미 에이전트가 로그를 발생시키고, 발생된 로그를 로컬 저장소에 저장합니다. 

* 더미 에이전트 로그 생성
  - lgde.info 와 lgde.debug 2가지 이벤트가 발생
  - info 이벤트만 filter, match 되도록 구성
  - debug 이벤트는 stdout 출력만 되도록 구성
* 필터 플러그인 구성
  - `record_transformer`를 통해 별도의 필트를 추가합니다
  - `${tag_parts[0]}` : 태그의 0번째 즉 lgde 를 말합니다
  - `record` : 앨리먼트를 통해서 `table_name`:`lgde` 라는 값이 추가됩니다
* 매치 플러그인 구성
  - 지정된 경로에 log 파일로 생성합니다
  - 1분에 한 번 저장하되 10초 까지 지연된 로그를 저장합니다
  - 한국 시간 기준으로 포맷을 생성 유지합니다

<br>


### 3-2. 에이전트 기동 및 확인

#### 3-2-1. 에이전트 기동을 위해 컨테이너로 접속 후, 에이전트를 기동합니다

```bash
# terminal
docker-compose exec fluentd bash
```
```bash
# docker
fluentd
```
* 아래와 같이 출력되고 있다면 정상입니다 (lgde.debug)
```bash
# docker
2021-07-18 11:52:24 +0000 [info]: #0 starting fluentd worker pid=58 ppid=49 worker=0
2021-07-18 11:52:24 +0000 [info]: #0 fluentd worker is now running worker=0
2021-07-18 11:52:25.081705098 +0000 lgde.debug: {"debug":"hello-world"}
2021-07-18 11:52:25.081710640 +0000 lgde.debug: {"debug":"hello-world"}
```
<br>


#### 3-2-2. 별도의 터미널에서 생성되는 로그 파일을 확인합니다

```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex2
docker-compose exec fluentd bash
```
```bash
# docker
tree /fluentd/target/
```

<details><summary> :green_book: 3. [기본] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```text

root@7d33f313cc13:~# 
/fluentd/target/
├── ${table_name}
│   └── %Y%m%d
│       └── part-%Y%m%d.%H%M
│           ├── buffer.b5c7647447f29a8c135e1164e39113f3b.log
│           └── buffer.b5c7647447f29a8c135e1164e39113f3b.log.meta
└── lgde
    └── 20210718
        └── part-20210718.2051_0.log

5 directories, 3 files
```

</details>
<br>

### 3-3. 에이전트 및 컨테이너 종료

#### 3-3-1. 실습이 끝났으므로 플루언트디 에이전트를 컨테이너 화면에서 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 종료합니다

#### 3-3-2. 1번 예제 실습이 모두 종료되었으므로 <kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 혹은 <kbd>exit</kbd> 명령으로 컨테이너를 종료합니다


[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>



## 4. 서버에 남는 로그를 지속적으로 모니터링하기

> 앞의 예제에서는 로그가 자동으로 생성된다는 것을 가정했는데, 이번 예제에서는 실제 로그가 생성되는 것을 재현해보고, 임의의 아파치 웹서버로그가 생성되는 것을 잘 모니터링하면서 수집하는 지를 실습합니다

![ex3](images/ex3.png)


### 4-1. 이전 컨테이너 종료 및 컨테이너 기동

> 이전 실습에서 기동된 컨테이너를 종료 후, 기동합니다.

```bash
cd ~/work/data-engineer-advanced-training/day2/ex2
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex3
docker-compose pull
docker-compose up -d
```
<br>


#### 4-1-1 도커 컴포즈 파일 구성 `docker-compose.yml`

```yaml
version: "3"

services:
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    volumes:
      - ./apache_logs:/home/root/apache_logs
      - ./flush_logs.py:/home/root/flush_logs.py
      - ./fluent.conf:/etc/fluentd/fluent.conf
      - ./source:/fluentd/source
      - ./target:/fluentd/target
    working_dir: /home/root

networks:
  default:
    name: day2_ex3_network
```
> 아파치 웹서버의 로그가 생성될 source 경로와 수집된 로그가 저장될 target 경로를 호스트 경로에 마운트 되었습니다.

<br>

#### 4-1-2 플루언트디 파일 구성 `fluent.conf`

```xml
<source>
    @type tail
    @log_level info
    path /fluentd/source/accesslogs
    pos_file /fluentd/source/accesslogs.pos
    refresh_interval 5
    multiline_flush_interval 5
    rotate_wait 5
    open_on_every_update true
    emit_unmatched_lines true
    read_from_head false
    tag weblog.info
    <parse>
        @type apache2
    </parse>
</source>

<match weblog.info>
    @type file
    @log_level info
    add_path_suffix true
    path_suffix .log
    path /fluentd/target/${tag}/%Y%m%d/accesslog.%Y%m%d.%H
    <buffer time,tag>
        timekey 1h
        timekey_use_utc false
        timekey_wait 10s
        timekey_zone +0900
        flush_mode immediate
        flush_thread_count 8
    </buffer>
</match>
```
> 소스 경로에 저장되는 로그를 모니터링하여 타겟 경로에 저장합니다 

* 소스 플러그인 설정
  - 로그 레벨은 info 로 지정된 경로에서 tail 플러그인으로 모니터링합니다
  - weblog.info 태그를 이용하였으며, apache2 로그를 파싱합니다
  - 로그파일이 새로 작성(log-rotate) 되더라도 최대 5초간 대기합니다
* 매치 플러그인 설정
  - 즉각적인 확인을 위해 최대한 자주 플러시 하도록 설정 했습니다
  <br>


#### 4-1-3. 아파치 로그 생성기 (`flush_logs.py`) 코드를 분석합니다

> 아파치 웹서버의 로그를 실제와 유사하게 저장하고, 롤링까지 하게 만들기 위해서 별도의 파이썬 스크립트가 필요합니다.

```python
#!/usr/bin/env python
import sys, time, os, shutil

# 1. read apache_logs flush every 100 lines until 1000 lines
# 2. every 1000 lines file close & rename file with seq
# 3. create new accesslogs and goto 1.

def readlines(fp, num_of_lines):
    lines = ""
    for line in fp:
        lines += line
        num_of_lines = num_of_lines - 1
        if num_of_lines == 0:
            break
    return lines

fr = open("apache_logs", "r")
for x in range(0, 10):
    fw = open("source/accesslogs", "w+")
    for y in range(0, 10):
        lines = readlines(fr, 100)
        fw.write(lines)
        fw.flush()
        time.sleep(0.1)
        sys.stdout.write(".")
        sys.stdout.flush()
    fw.close()
    print("file flushed ... sleep 10 secs")
    time.sleep(10)
    shutil.move("source/accesslogs", "source/accesslogs.%d" % x)
    print("renamed accesslogs.%d" % x)
fr.close()
```
> 원본 로그를 읽어서 source 경로에 저장하는 스크립트

* 한 번에 100 라인씩을 읽어서 source 의 accesslogs 에 flush 합니다
* 매 1000라인 마다 마치 용량이 커서 롤링된 것처럼 파일명을 변경합니다
* 롤링된 이후에는 원본 accesslogs 파일에 계속 append 합니다
<br>


### 4-2. 에이전트 기동 및 확인

#### 4-2-1. 에이전트 기동을 위해 컨테이너로 접속 후, 에이전트를 기동합니다

```bash
# terminal
docker-compose exec fluentd bash
```
```bash
# docker
fluentd
```
> 플루언트디의 경우 기동 시에 오류가 없었다면 정상적으로 기동 되었다고 보시면 됩니다

<br>

#### 4-2-2. 시스템 로그를 임의로 생성

> 웹서버의 로그를 생성하기 어렵기 때문에 기존에 적재된 액세스로그를 읽어서 주기적으로 accesslog 파일에 append 하는 프로그램(`flush_logs.py`)을 통해서 시뮬레이션 합니다

* 로그 생성기를 통해 accesslog 파일을 계속 source 경로에 append 하고, target 경로에서는 수집되는지 확인합니다

```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex3
docker-compose exec fluentd bash
```
```bash
python flush_logs.py
```
<br>


* 별도의 터미널을 통해 로그가 생성되는지 확인합니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex3
docker-compose exec fluentd bash
```
```
for x in $(seq 1 100); do tree -L 1 /fluentd/source; tree -L 2 /fluentd/target; sleep 10; done
```

> 위의 명령어로 주기적으로 출력 경로를 확인할 수 있습니다

<details><summary> :green_book: 4. [기본] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```bash
021-07-18 13:01:20 +0000 [info]: #0 detected rotation of /fluentd/source/accesslogs
2021-07-18 13:01:20 +0000 [info]: #0 following tail of /fluentd/source/accesslogs
2021-07-18 13:01:31 +0000 [info]: #0 detected rotation of /fluentd/source/accesslogs
2021-07-18 13:01:31 +0000 [info]: #0 following tail of /fluentd/source/accesslogs
2021-07-18 13:01:32 +0000 [warn]: #0 pattern not matched: "46.118.127.106 - - [20/May/2015:12:05:17 +0000] \"GET /scripts/grok-py-test/configlib.py HTTP/1.1\" 200 235 \"-\" \"Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html"
2021-07-18 13:01:43 +0000 [info]: #0 detected rotation of /fluentd/source/accesslogs
2021-07-18 13:01:43 +0000 [info]: #0 following tail of /fluentd/source/accesslogs
2021-07-18 13:01:54 +0000 [info]: #0 detected rotation of /fluentd/source/accesslogs; waiting 5.0 seconds
```

```bash
root@2cf7c79e8367:~# for x in $(seq 1 100); do tree -L 1 /fluentd/source; tree -L 2 /fluentd/target; sleep 10; done
/fluentd/source
├── accesslogs.0
├── accesslogs.1
├── accesslogs.2
├── accesslogs.3
├── accesslogs.4
├── accesslogs.5
├── accesslogs.6
├── accesslogs.7
├── accesslogs.8
├── accesslogs.9
└── accesslogs.pos

0 directories, 11 files
/fluentd/target
├── ${tag}
│   └── %Y%m%d
└── weblog.info
    ├── 20150517
    ├── 20150518
    ├── 20150519
    ├── 20150520
    ├── 20150521
    └── 20210718
```

</details>
<br>

### 4-3. 에이전트 및 컨테이너 종료

#### 4-3-1. 실습이 끝났으므로 플루언트디 에이전트를 컨테이너 화면에서 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 종료합니다

#### 4-3-2. 1번 예제 실습이 모두 종료되었으므로 <kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 혹은 <kbd>exit</kbd> 명령으로 컨테이너를 종료합니다


### 4-4. 컨테이너 정리
* 테스트 작업이 완료되었으므로 모든 컨테이너를 종료합니다 (한번에 실행중인 모든 컨테이너를 종료합니다)
```bash
cd ~/work/data-engineer-advanced-training/day2/ex3
docker-compose down
```

[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>


## 2. 파일 수집 고급

## 5. 데이터 변환 플러그인을 통한 시간 변경 예제

> 로그 수집 및 적재 시에 가장 민감한 부분이 시간이며, 이에 따라 지연, 유실 등의 다양한 문제의 원인이 되기도 합니다. 원본 로그에는 Unix Timestamp 형식의 로그로 저장되는 것이 일반적이므로 이를 어떻게 수신, 분석 및 저장하는 지에 대한 실습을 합니다


### 5-1. 이전 컨테이너 종료 및 컨테이너 기동

> 이전 실습에서 기동된 컨테이너를 종료 후, 기동합니다.

```bash
cd ~/work/data-engineer-advanced-training/day2/ex3
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex4
docker-compose pull
docker-compose up -d
```
<br>


#### 5-1-1 도커 컴포즈 파일 구성 `docker-compose.yml`

```yaml
version: "3"

services:
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    ports:
      - 8080:8080
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
    working_dir: /home/root

networks:
  default:
    name: day2_ex4_network

```
> 시간 정보를 담은 JSON 을 전송할 포트를 8080 으로 정했습니다

<br>

#### 5-1-2 플루언트디 파일 구성 `fluent.conf`

```xml
<source>
    @type http
    port 8080
    <parse>
        @type json
        time_type float
        time_key logtime
        types column1:integer,column2:string,logtime:time:unixtime
        localtime true
        keep_time_key true
    </parse>
</source>

<filter lgde>
    @type record_transformer
    enable_ruby
    <record>
        filtered_logtime ${Time.at(time).strftime('%Y-%m-%d %H:%M:%S')}
    </record>
</filter>

<match lgde>
    @type stdout
    <format>
        @type json
        time_format %Y-%m-%d %H:%M:%S.%L
        timezone +09:00
    </format>
</match>
```
> HTTP 서버를 통해 JSON 수신 및 변환 이후에 화면에 출력합니다

* 소스 HTTP 플러그인 구성
  - 입력 데이터의 포맷은 column1, column2, logtime 이며 JSON 형식입니다 
  - 각 컬럼별 데이터 타입을 지정 할 수 있으며 time 컬럼은 logtime 입니다
  - 기존의 시간 컬럼은 유지합니다
* 필터 및 매치 플러그인 구성
  - lgde 태그를 통해 전송되는 로그에 대해서 '2021-07-18 22:28:00' 포맷으로 변환합니다
  - `filtered_logtime` 컬럼이 추가됩니다
  - 출력 포맷 또한 JSON 이며 현재 시간을 기준으로 출력됩니다

<br>

### 5-2. 에이전트 기동 및 확인

#### 5-2-1. 에이전트 기동을 위해 컨테이너로 접속 후, 에이전트를 기동합니다

```bash
# terminal
docker-compose exec fluentd bash
```
```bash
# docker
fluentd
```
<br>


#### 5-2-2. 컨테이너에 접속하여 시간을 포함한 JSON 을 POST 로 전송합니다

> Epoch 시간은 https://www.epochconverter.com/ 사이트를 활용합니다

* 별도로 컨테이너에 접속하여 예제로 현재 시간을 넣고 로그를 출력합니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex4
docker-compose exec fluentd bash
```
<br>

```bash
# docker
curl -X POST -d '{ "column1":"1", "column2":"hello-world", "logtime": 1593379470 }' http://localhost:8080/lgde
```
<br>


#### 5-2-3. 1시간 단위로 시간을 늘려가며 전송하는 테스트를 해봅니다

> seq 라는 커맨드라인을 이용하여 `2021-07-17 ~ 2021-08-17` 까지 1시간(3600초)씩 늘려가면서 로그를 전송해봅니다

* bash 에서 실행 시에 JSON 이 POST 데이터로 제대로 넘어가기 위해서는 escape 문자(back-slash`\`)를 이용해야 합니다

```bash
for x in $(seq 1626479572 3600 1726479572); do \
    curl -X POST -d "{ \"column1\":\"1\", \"column2\":\"hello-world\", \"logtime\": \"$x\" }" \
    http://localhost:8080/lgde; \
    sleep 1; \
done
```

<details><summary> :blue_book: 5. [중급] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

```bash
2021-07-18 13:22:39 +0000 [info]: starting fluentd-1.11.5 pid=196 ruby="2.4.10"
2021-07-18 13:22:39 +0000 [info]: spawn command to main:  cmdline=["/opt/td-agent/embedded/bin/ruby", "-Eascii-8bit:ascii-8bit", "/opt/td-agent/embedded/bin/fluentd", "-c", "/etc/fluentd/fluent.conf", "--under-supervisor"]
2021-07-18 13:22:39 +0000 [info]: adding filter pattern="test" type="record_transformer"
2021-07-18 13:22:40 +0000 [info]: adding match pattern="test" type="stdout"
2021-07-18 13:22:40 +0000 [info]: adding source type="http"
2021-07-18 13:22:40 +0000 [info]: #0 starting fluentd worker pid=205 ppid=196 worker=0
2021-07-18 13:22:40 +0000 [info]: #0 fluentd worker is now running worker=0
{"column1":1,"column2":"hello-world","logtime":1626479572,"filtered_logtime":"2021-07-16 23:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626483172,"filtered_logtime":"2021-07-17 00:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626486772,"filtered_logtime":"2021-07-17 01:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626490372,"filtered_logtime":"2021-07-17 02:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626493972,"filtered_logtime":"2021-07-17 03:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626497572,"filtered_logtime":"2021-07-17 04:52:52"}
{"column1":1,"column2":"hello-world","logtime":1626501172,"filtered_logtime":"2021-07-17 05:52:52"}
```

</details>
<br>


### 5-3. 에이전트 및 컨테이너 종료

#### 5-3-1. 실습이 끝났으므로 플루언트디 에이전트를 컨테이너 화면에서 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 종료합니다

#### 5-3-2. 1번 예제 실습이 모두 종료되었으므로 <kbd><samp>Ctrl</samp>+<samp>D</samp></kbd> 혹은 <kbd>exit</kbd> 명령으로 컨테이너를 종료합니다


[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>


## 6. 컨테이너 환경에서의 로그 전송

> 클라우드 환경에서의 애플리케이션 들은 컨테이너 환경에서 운영되기 때문에 **로컬 스토리지를 이용해서 로그를 저장하는 경우 예기치 않은 컨테이너 종료 시에는 로그 유실**을 피할 수 없게 됩니다. 하여 *도커 컨테이너와 같이 플루언트디와 연동된 플러그인을 지원*합니다. 도커의 경우 Log Driver 형태로 플루언트디를 지원하며 기동 시에 설정을 하게 되면 **유실 없는 로그 저장 및 관리가 가능**합니다

* 임의의 로그를 출력하는 도커 컨테이너(ubuntu)를 기동 시킵니다
* 해당 컨테이너 기동 시에 플루언트디로 전송설정을 합니다
* 설정된 로그 수집기(fluent.conf)를 통해 로그를 출력합니다
* 하나의 애플리케이션과 전송 에이전트가 하나의 셋트로 배포됩니다
  - **결국 다른 수집기(aggregator)로 전송하여 전체 로그를 집계**하게 됩니다

<br>

### 6-1. 이전 컨테이너 종료 및 컨테이너 기동

> 이전 실습에서 기동된 컨테이너를 종료 후, 기동합니다.

```bash
cd ~/work/data-engineer-advanced-training/day2/ex4
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex5
docker-compose pull
docker-compose up -d
```
<br>


#### 6-1-1 도커 컴포즈 파일 구성 `docker-compose.yml`

```yaml
version: '3'

services:
  ubuntu:
    container_name: ubuntu
    image: psyoblade/data-engineer-ubuntu:20.04
    stdin_open: true
    tty: true
    restart: always
    volumes:
      - ./fortune_teller.sh:/fortune_teller.sh
    entrypoint: [ "/fortune_teller.sh", "1000" ]
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-async: "true"
        fluentd-address: localhost:24224
        tag: docker.fortune
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    ports:
      - 24224:24224
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
    entrypoint: [ "/home/root/fluentd" ]
    working_dir: /home/root

networks:
  default:
    name: day2_ex5_network
```
> 로그를 생성하는 ubuntu 컨테이너와 로그를 수신하는 fluentd 컨테이너를 기동합니다

* ubuntu 기반 애플리케이션을 entrypoint 통해 수행됩니다
* fortune 명령을 1000회 반복하는 `fortune_teller.sh` 을 기동
* logging 설정을 통해 docker.fortune 태그를 생성하여 전송합니다
* fluentd 에이전트 또한 entrypoint 통해 같이 수행 되어야 합니다
* fluentd 에이전트가 기동되기 전에 접근을 하면 오류가 발생하므로 `fluentd-async: "true"` 옵션을 통해 비동기 접속을 설정합니다

<br>

#### 6-1-2 플루언트디 파일 구성 `fluent.conf`

```xml
<source>
    @type forward
    port 24224
    bind 0.0.0.0
</source>

<filter docker.*>
    @type parser
    key_name log
    reserve_data true
    <parse>
        @type json
    </parse>
</filter>

<filter docker.*>
    @type record_transformer
    <record>
        table_name ${tag_parts[1]}
    </record>
</filter>

<match docker.*>
    @type stdout
</match>
```
> 기본 플루언트디 포트(24224)로 전송된 이벤트들을 표준 출력으로 내보냅니다

* 필터 플러그인 설정
  - docker 로 시작하는 태그를 모두 필터하고, JSON 포맷만 전달합니다
  - dokcer 로 시작하는 태그 이벤트를 필터하여 docker.{} 에 해당하는 레코드를 추가합니다

<br>

#### 6-1-3 포츈텔러 파일 구성 `fortune_teller.sh`

```bash
#!/bin/bash
sleep 3
for x in $(seq 1 $1); do 
    say=`fortune -s`
    out=`echo ${say@Q} | sed -e "s/'//g" | sed -e 's/"//g' | sed -e 's/\\\\//g'`
    echo "{\"key\":\"$out\"}"
    sleep 1
done
```
> 3초 대기 후 fortune 명령을 1초 간격으로 입력된 횟수만큼(1,000)회 반복합니다출력합니다

<br>


### 6-2. 에이전트 기동 및 확인

#### 6-2-1. 에이전트 기동을 위해 컨테이너로 접속 후, 에이전트를 기동합니다

```bash
# terminal
docker-compose up -d
```
```bash
docker-compose logs -f ubuntu
```
> 애플리케이션에서 로그가 잘 전송되고 있는지 확인합니다

<br>

```bash
cd ~/work/data-engineer-advanced-training/day2/ex5
docker-compose logs -f fluentd
```
> 플루언트디로 전송되어 출력이 되는지 확인합니다

<details><summary> :blue_book: 6. [중급] 출력 결과 확인</summary>

> 출력 결과가 오류가 발생하지 않고, 아래와 유사하다면 성공입니다

* 애플리케이션 로그
```bash
ubuntu  | {"key":"You will gain money by an illegal action."}
ubuntu  | {"key":"$Q:tWhats the difference between Bell Labs and the Boy Scouts of America?nA:tThe Boy Scouts have adult supervision."}
ubuntu  | {"key":"$Its a very *__bbUN*lucky week in which to be took dead.ntt-- Churchy La Femme"}
ubuntu  | {"key":"$Many pages make a thick book, except for pocket Bibles which are on verynvery thin paper."}
ubuntu  | {"key":"Your heart is pure, and your mind clear, and your soul devout."}
ubuntu  | {"key":"$Q:tHow many lawyers does it take to change a light bulb?nA:tOne. Only its his light bulb when hes done."}
ubuntu  | {"key":"$FORTUNE PROVIDES QUESTIONS FOR THE GREAT ANSWERS: #21nA:tDr. Livingston I. Presume.nQ:tWhats Dr. Presumes full name?"}
ubuntu  | {"key":"Snow Day -- stay home."}

```

* 플루언트디 로그
```bash
fluentd  | 2021-07-18 17:03:09.161100283 +0000 docker.fortune: {"source":"stdout","log":"{\"key\":\"Executive ability is prominent in your make-up.\"}\r","container_id":"af6051b411773b40623f57203e85054a085af57d02097e083dd1e7aade854484","container_name":"/ubuntu","key":"Executive ability is prominent in your make-up.","table_name":"fortune"}
fluentd  | 2021-07-18 17:03:10.170363442 +0000 docker.fortune: {"log":"{\"key\":\"$Q:tWhat do you call a half-dozen Indians with Asian flu?nA:tSix sick Sikhs (sic).\"}\r","container_id":"af6051b411773b40623f57203e85054a085af57d02097e083dd1e7aade854484","container_name":"/ubuntu","source":"stdout","key":"$Q:tWhat do you call a half-dozen Indians with Asian flu?nA:tSix sick Sikhs (sic).","table_name":"fortune"}
fluentd  | 2021-07-18 17:03:11.177728881 +0000 docker.fortune: {"container_id":"af6051b411773b40623f57203e85054a085af57d02097e083dd1e7aade854484","container_name":"/ubuntu","source":"stdout","log":"{\"key\":\"$Fine day for friends.nSo-so day for you.\"}\r","key":"$Fine day for friends.nSo-so day for you.","table_name":"fortune"}
```

</details>
<br>

#### 6-2-2. 컨테이너를 통해 전달할 수 있는 파라메터

* [Customize log driver output](https://docs.docker.com/config/containers/logging/log_tags/) 참고하여 작성 되었습니다

| Markup | Description |
| --- | --- |
| {{.ID}} | The first 12 characters of the container ID. |
| {{.FullID}} | The full container ID. |
| {{.Name}} | The container name. |
| {{.ImageID}} | The first 12 characters of the container’s image ID. |
| {{.ImageFullID}} | The container’s full image ID. |
| {{.ImageName}} | The name of the image used by the container. |
| {{.DaemonName}} | The name of the docker program (docker). |

<details><summary> :blue_book: 7. [중급] 기존 `docker-compose.yml` 파일을 활용하여, ubuntu 서비스를 web 으로 변경하여 새롭게 복사하고 tag 를 `docker.{{.Name}}}`로 출력하는 `web.yml`을 작성하고 기동하여, ubuntu, web, fluentd 이렇게 3개의 컨테이너 서비스가 운영되는 web.yml 파일을 생성하고 기동합니다 </summary>


> `web.yml` 파일을 아래와 같이 작성하셨다면 정답입니다

```yaml
version: '3'

services:
  ubuntu:
    container_name: ubuntu
    image: psyoblade/data-engineer-ubuntu:20.04
    stdin_open: true
    tty: true
    restart: always
    volumes:
      - ./fortune_teller.sh:/fortune_teller.sh
    entrypoint: [ "/fortune_teller.sh", "1000" ]
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: docker.fortune
  web:
    container_name: web
    image: psyoblade/data-engineer-ubuntu:20.04
    stdin_open: true
    tty: true
    restart: always
    volumes:
      - ./fortune_teller.sh:/fortune_teller.sh
    entrypoint: [ "/fortune_teller.sh", "1000" ]
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: docker.{{.Name}}
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    stdin_open: true
    tty: true
    ports:
      - 24224:24224
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
    entrypoint: [ "/home/root/fluentd" ]
    working_dir: /home/root

networks:
  default:
    name: day2_ex5_web_network
```

* 기존에 떠 있던 컨테이너를 종료하고 `web.yml`을 기동합니다
```bash
docker-compose down
docker-compose -f web.yml up -d
```

* 개별 서비스들의 로그를 하나 하나 확인해 봅니다
```bash
docker-compose -f web.yml logs -f ubuntu
docker-compose -f web.yml logs -f web
docker-compose -f web.yml logs -f fluentd
```

* 미처 종료되지 않은 컨테이너도 종료합니다
```bash
# terminal
docker-compose -f web.yml down --remove-orphans
```

</details>


[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>

## 7. 도커 컴포즈를 통한 로그 전송 구성

### 7-1. 도커 로그 수집 컨테이너를 기동하고, web, kibana, elasticsearch 모두 떠 있는지 확인합니다

```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex5
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex6
docker-compose pull
docker-compose up -d
docker ps
```

### 7-2. 키바나를 통해 엘라스틱 서치를 구성합니다

* 키바나 사이트에 접속하여 색인을 생성
  * 1. `http://my-cloud.host.com:8080` 사이트에 접속 (모든 컴포넌트 기동에 약 3~5분 정도 소요됨)
  * 2. Explorer on my Own 선택 후, 좌측 "Discover" 메뉴 선택
  * 3. Step1 of 2: Create Index with 'fluentd-\*' 까지 치면 아래에 색인이 뜨고 "Next step" 클릭
  * 4. Step2 of 2: Configure settings 에서 @timestamp 필드를 선택하고 "Create index pattern" 클릭
  * 5. Discover 메뉴로 이동하면 전송되는 로그를 실시간으로 확인할 수 있음

* 웹 사이트에 접속을 시도 (초기 기동 시에 시간이 걸립니다)
  * 1. `http://my-cloud.host.com` 혹은 `localhost`에 접속하여 It works! 가 뜨면 정상입니다
  * 2. 다시 Kibana 에서 Refresh 버튼을 누르면 접속 로그가 전송됨을 확인

### 7-3. Fluentd 구성 파일 `docker-compose.yml`

```yaml
version: "3"
services:
  web:
    container_name: web
    image: httpd
    ports:
      - 80:80
    links:
      - fluentd
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: httpd.access
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd
    volumes:
      - ./fluentd/fluent.conf:/fluentd/etc/fluent.conf
    links:
      - elasticsearch
    ports:
      - 24224:24224
      - 24224:24224/udp
    entrypoint: [ "fluentd" ]
  elasticsearch:
    container_name: elasticsearch
    image: elasticsearch:7.8.0
    command: elasticsearch
    environment:
      - cluster.name=demo-es
      - discovery.type=single-node
      - http.cors.enabled=true
      - http.cors.allow-credentials=true
      - http.cors.allow-headers=X-Requested-With,X-Auth-Token,Content-Type,Content-Length,Authorization
      - http.cors.allow-origin=/https?:\/\/localhost(:[0-9]+)?/
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    expose:
      - 9200
    ports:
      - 9200:9200
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
      - ./elasticsearch/logs:/usr/share/elasticsearch/logs
    healthcheck:
      test: curl -s https://localhost:9200 >/dev/null; if [[ $$? == 52 ]]; then echo 0; else echo 1; fi
      interval: 30s
      timeout: 10s
      retries: 5
  kibana:
    container_name: kibana
    image: kibana:7.8.0
    links:
      - elasticsearch
    ports:
      - 8080:5601

networks:
  default:
    name: day2_ex6_network
```

[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>

## 8. 멀티 프로세스를 통한 성능 향상

### 8-1. 서비스를 기동하고 별도의 터미널을 통해서 멀티프로세스 기능을 확인합니다 (반드시 source/target 경로를 호스트에서 생성합니다)

* `docker` 가 경로를 생성하면 root 권한으로 생성되어 `fluentd`가 파일저장에 실패할 수 있습니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex6
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex7
mkdir source
mkdir target
docker-compose up -d
docker-compose logs -f
```
<br>

### 8-2. `docker-compose.yml` 파일 구성
```yml
version: "3"

services:
  fluentd:
    container_name: multi-process
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
      - ./source:/fluentd/source
      - ./target:/fluentd/target
    ports:
      - 9880:9880
      - 24224:24224
      - 24224:24224/udp
    entrypoint: [ "fluentd" ]

networks:
  defualt:
    name: day2_ex7_network
```

### 8-3. `fluentd.conf` 파일 구성
```html
<system>
    workers 2
    root_dir /fluentd/log
</system>
<worker 0>
    <source>
        @type tail
        path /fluentd/source/*.log
        pos_file /fluentd/source/local.pos
        tag worker.tail
        <parse>
            @type json
        </parse>
    </source>
    <match>
        @type stdout
    </match>
</worker>
<worker 1>
    <source>
        @type http
        port 9880
        bind 0.0.0.0
    </source>
    <match>
        @type stdout
    </match>
</worker>
```

### 8-4. 새로운 터미널에서 다시 아래의 명령으로 2가지 테스트를 수행합니다.

* 첫 번째 프로세스가 파일로 받은 입력을 표준 출력으로 내보내는 프로세스입니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex7
for x in $(seq 1 1000); do sleep 0.1 ; echo "{\"hello\":\"world\"}" >> source/start.log; done
```
* 두 번째 프로세스는 HTTP 로 입력 받은 내용을 표준 출력으로 내보내는 프로세스입니다
```bash
cd ~/work/data-engineer-advanced-training/day2/ex7
curl -XPOST -d "json={\"hello\":\"world\"}" http://localhost:9880/test
```
<br>

[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>

## 9. 멀티 프로세스를 통해 하나의 위치에 저장

### 9-1. 서비스를 기동하고 별도의 터미널을 통해서 멀티프로세스 기능을 확인합니다 (반드시 source/target 경로를 호스트에서 생성합니다)
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex7
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex8
mkdir source
mkdir target
docker-compose up -d
docker-compose logs -f
```
<br>

### 9-2. `docker-compose.yml` 파일 구성
```yml
version: "3"

services:
  fluentd:
    container_name: multi-process-ex
    image: psyoblade/data-engineer-fluentd:2.2
    user: root
    volumes:
      - ./fluent.conf:/etc/fluentd/fluent.conf
      - ./source:/fluentd/source
      - ./target:/fluentd/target
    ports:
      - 9880:9880
      - 9881:9881
      - 24224:24224
      - 24224:24224/udp
    networks:
      - default
    entrypoint: [ "fluentd" ]

networks:
  default:
    name: day2_ex8_network
```
<br>

### 9-3. Fluentd 구성 파일을 분석합니다

* `fuent.conf`
```yaml
<system>
    workers 2
    root_dir /fluentd/log
</system>

<worker 0>
    <source>
        @type http
        port 9880
        bind 0.0.0.0
    </source>
    <match debug>
        @type stdout
    </match>
    <match test>
        @type file
        @id out_file_0
        path "/fluentd/target/#{worker_id}/${tag}/%Y/%m/%d/testlog.%H%M"
        <buffer time,tag>
            timekey 1m
            timekey_use_utc false
            timekey_wait 10s
        </buffer>
    </match>
</worker>

<worker 1>
    <source>
        @type http
        port 9881
        bind 0.0.0.0
    </source>
    <match debug>
        @type stdout
    </match>
    <match test>
        @type file
        @id out_file_1
        path "/fluentd/target/#{worker_id}/${tag}/%Y/%m/%d/testlog.%H%M"
        <buffer time,tag>
            timekey 1m
            timekey_use_utc false
            timekey_wait 10s
        </buffer>
    </match>
</worker>
```
<br>

### 9-4. 별도의 터미널에서 아래의 명령을 수행합니다

* 멀티 프로세스를 통해 하나의 경로에 저장되는 것을 확인합니다

```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex8
tree target
```
<br>

* `progress.sh`
```bash
#!/bin/bash
max=120
dot="."
for number in $(seq 0 $max); do
    for port in $(seq 9880 9881); do
        # echo curl -XPOST -d "json={\"hello\":\"$number\"}" http://localhost:$port/test
        curl -XPOST -d "json={\"hello\":\"$number\"}" http://localhost:$port/test
    done
    echo -ne "[$number/$max] $dot\r"
    sleep 1
    dot="$dot."
done
echo
tree
```

<br>


[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>


## 10. 전송되는 데이터를 분산 저장소에 저장

### 10-1. 서비스를 기동합니다
```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex8
docker-compose down

cd ~/work/data-engineer-advanced-training/day2/ex9
docker-compose up -d
docker-compose logs -f
```
<br>

### 10-2. `docker-compose.yml` 파일을 분석합니다
```yml
version: "3"

services:
  fluentd:
    container_name: fluentd
    image: psyoblade/data-engineer-fluentd:2.2
    depends_on:
      - namenode
      - datanode
    user: root
    stdin_open: true
    tty: true
    volumes:
      - ./fluentd/fluent.conf:/etc/fluentd/fluent.conf
    ports:
      - 9880:9880
    networks:
      - default
    entrypoint: [ "fluentd" ]
...
```

### 10-3. `fluent.conf` 파일을 분석합니다
```xml
<source>
    @type http
    port 9880
    bind 0.0.0.0
</source>

<match test.info>
    @type webhdfs
    @log_level debug
    host namenode
    port 50070
    path "/user/fluent/webhdfs/logs.${tag}.%Y%m%d_%H.#{Socket.gethostname}.log"
    <buffer time,tag>
        timekey 1m
        timekey_wait 10s
        flush_interval 30s
    </buffer>
</match>

<match test.debug>
    @type stdout
</match>
```

### 10-4. 별도의 터미널에서 모든 서비스(fluentd, namenode, datanode)가 떠 있는지 확인합니다

```bash
# terminal
cd ~/work/data-engineer-advanced-training/day2/ex9
docker ps
```
<br>

### 10-5. HTTP 로 전송하고 해당 데이터가 하둡에 저장되는지 확인합니다

> http://my-cloud.host.com:50070/explorer.html 에 접속하여 확인하거나, namenode 에 설치된 hadoop client 로 확인 합니다

* WARN: 현재 노드수가 1개밖에 없어서 발생하는 Replication 오류는 무시해도 됩니다
  - 네임노드가 정상 기동될 때까지 약 30초 정도 대기합니다
```bash
cd ~/work/data-engineer-advanced-training/day2/ex9
sleep 30
./progress.sh
```

* `progress.sh` : 테스트를 위한 로그를 전송합니다
```bash
#!/bin/bash
max=120
dot="."
for number in $(seq 0 $max); do
    curl -XPOST -d "json={\"hello\":\"$number\"}" http://localhost:9880/test.info
    echo -ne "[$number/$max] $dot\r"
    sleep 1
    dot="$dot."
done
```

* 1분 이상 대기했다가 확인합니다
```bash
docker exec -it namenode hadoop fs -ls /user/fluent/webhdfs/
```
<br>

### 10-5. 실습이 끝났으므로 플루언트디 에이전트를 컨테이너 화면에서 <kbd><samp>Ctrl</samp>+<samp>C</samp></kbd> 명령으로 종료합니다

```bash
# terminal
docker-compose down
```

</details>
<br>

[목차로 돌아가기](#2일차-스트림-데이터-수집)

<br>
<br>

# 3. 연습문제

>  모든 퀴즈는 기존의 컨테이너 들은 모두 종료 되어 있어야 정상 동작하며, 반드시 `quiz<number>` 경로로 이동 후,  `docker-compose.yml` 파일을 이용하여 컨테이너 기동 후, `fluentd` 명령으로 수행하여 퀴즈를 푸시면 됩니다.

## 3-1. 참고 사이트

* [Parse data  types](https://docs.fluentd.org/configuration/parse-section#types-parameter)

* [Record transformer](https://docs.fluentd.org/filter/record_transformer)

* ```bash
  # cURL 명령어 통한 POST 전송 예제
  # -i, --include : 프로토콜 반환 값의 헤더를 출력합니다
  # -X, --request <command> : 리퀘스트 타입을 지정합니다
  # -d, --data <data> : HTTP POST 데이터
  
  curl -i -X POST -d '{"column1":"1","column2":"hello-world","column3":20220722,"logtime":1593379470}' http://localhost:8080/lgde
  ```

* [Customize log driver output](https://docs.docker.com/config/containers/logging/log_tags/)
* [Docker run reference](https://docs.docker.com/engine/reference/run/)

* ```bash
  # docker run 명령을 통한 컨테이너 1회성 실행
  # --rm : 컨테이너 종료시에 관련 파일 시스템 및 메타 정보를 같이 삭제합니다
  # --name container_name : 컨테이너의 이름을 지정합니다
  # --log-driver=driver_name : 도커 컨테이너의 로그 드라이버를 지정
  # --log-opt key=value : 로그 드라이버에 전달하는 추가 옵션을 지정
  
  docker run --rm --name 'container_name' --log-driver=fluentd --log-opt tag=tag.markup ubuntu echo '{"message":"send message with name"}' 
  ```

<br>

## 3-2. 퀴즈

>  `fluentd`의 기본 동작 방식을 이해하고, 데이터 소스와 싱크의 동작 방식에 대한 이해를 전제로 하고 있습니다. 잘 이해가 안 가거나 어렵다고 생각되시면 관련 챕터로 이동하여 다시 한 번 기존 예제를 리뷰해 보시면 도움이 되실 겁니다.

<details><summary> :blue_book: Quiz 1. 컨테이너를 기동하고, 'send_http.sh' 실행 시에 발생하는 오류를 해결하세요</summary>


>  `docker-compose logs -f fluentd` 실행 결과에서 아래와 같이 메시지가 출력되면 성공입니다

```bash
2022-07-24 02:19:23.844319643 +0000 debug: {"action":"login","user":2}
```

> 아래와 유사하게 `send_http.sh` 파일을 수정하시면 됩니다

```bash
#!/bin/bash
curl -i -X POST -d 'json={"action":"login","user":2}' http://localhost:9881/debug
```

</details>

<br>

<details><summary> :closed_book: Quiz 2. 'fluent.conf' 파일을 수정하여 최종 출력 로그가 debug 태그는 target/debug/yyyyMM 경로에, info 태그는 target/info/yyyyMM 경로로 저장되도록 변경 후, fluentd 프로세스를 다시 시작해서 확인해 보세요</summary>

>  `docker-compose logs -f fluentd` 실행 결과에서 `tree target` 명령 결과가 아래와 유사하다면 성공입니다

```bash
target
├── ${table_name}
│   └── %Y%m
│       └── part-%Y%m%d.%H%M
│           ├── buffer.b5e483bf0275ce4fc70afdad5df8ffec9.log
│           ├── buffer.b5e483bf0275ce4fc70afdad5df8ffec9.log.meta
│           ├── buffer.b5e483bf0282f98cf6e06eb2e265b890f.log
│           └── buffer.b5e483bf0282f98cf6e06eb2e265b890f.log.meta
├── debug
│   └── 202207
│       └── part-20220724.1122_0.log
└── info
    └── 202207
        ├── part-20220724.1121_0.log
        └── part-20220724.1122_0.log
```

> 아래와 유사하게 `fluent.conf` 파일이 수정되면 됩니다

```xml
<source>
    @type dummy
    tag lgde.info
    size 5
    rate 1
    auto_increment_key seq
    dummy {"info":"hello-world"}
</source>

<source>
    @type dummy
    tag lgde.debug
    size 3
    rate 1
    dummy {"debug":"hello-world"}
</source>

<filter lgde.*>
    @type record_transformer
    <record>
        table_name ${tag_parts[1]}
    </record>
</filter>

<match lgde.*>
    @type file
    path_suffix .log
    path /fluentd/target/${table_name}/%Y%m/part-%Y%m%d.%H%M
    <buffer time,table_name>
        timekey 1m
        timekey_wait 10s
        timekey_use_utc false
        timekey_zone +0900
    </buffer>
</match>
```

</details>

<br>

<details><summary> :blue_book: Quiz 3. fluentd 프로세스를 기동하면 source 에 존재하는 모든 파일이 수집되어 더 이상 수집되지 않습니다. 누군가가 실수로 target 경로를 모두 삭제하여 장애가 발생하고 있습니다. 다행스럽게도 source 경로의 파일은 존재하여 다시 수집할 수 있는 상황입니다. 다시 원본 데이터를 target 경로에 수집할 수 있도록 장애 복구를 해주세요. </summary>

> 아래와 같이 수집 완료된 로그가 모두 1만 라인이면 성공입니다.

```bash
cat target/weblog.info/*/* | wc -l
10000
```

>  아래와 유사하게 `fluent.conf` 파일이 수정하고, `/fluentd/source/fluent.pos` 파일 삭제 후 `fluentd` 에이전트를 재기동 하면 됩니다

```xml
<source>
    @type tail
    @log_level info
    path /fluentd/source/accesslogs*
    pos_file /fluentd/source/fluent.pos
    refresh_interval 5
    multiline_flush_interval 5
    rotate_wait 5
    open_on_every_update true
    emit_unmatched_lines true
    read_from_head true
    tag weblog.info
    <parse>
        @type apache2
    </parse>
</source>

<match weblog.info>
    @type file
    @log_level info
    add_path_suffix true
    path_suffix .log
    path /fluentd/target/${tag}/%Y%m%d/accesslog.%Y%m%d.%H
    <buffer time,tag>
        timekey 1h
        timekey_use_utc false
        timekey_wait 10s
        timekey_zone +0900
        flush_mode immediate
        flush_thread_count 8
    </buffer>
</match>
```

</details>

<br>

<details><summary> :closed_book: Quiz 4. 초기에 전송되던 로그의 포맷이 변경되어 column2 다음에 column3 (integer) 가 추가되었다고 합니다. 'fluentd' 에이전트 로그에서 신규 컬럼이 출력되도록 'fluent.conf' 파일을 변경하고 fluentd 에이전트를 재기동하여 curl 명령으로 column3 이 추가된 데이터 전송을하고, `curl -X POST -d '{"column1":"1","column2":"hello-world","column3":20220722,"logtime":1593379470}' http://localhost:8080/lgde` 결과가 잘 나오는지 확인해 보세요
 </summary>

>  `docker-compose logs -f fluentd` 실행 결과에서 아래와 같이 메시지가 출력되면 성공입니다

```bash
{"column1":1,"column2":"hello-world","column3":20220722,"logtime":1593379470,"filtered_logtime":"2020-06-28 21:24:30"}
```

> `fluent.conf` 파일이 아래와 같으면 정답입니다

```xml
<source>
    @type http
    port 8080
    <parse>
        @type json
        time_type float
        time_key logtime
        types column1:integer,column2:string,column3:integer,logtime:time:unixtime
        localtime true
        keep_time_key true
    </parse>
</source>

<filter lgde>
    @type record_transformer
    enable_ruby
    <record>
        filtered_logtime ${Time.at(time).strftime('%Y-%m-%d %H:%M:%S')}
    </record>
</filter>

<match lgde>
    @type stdout
    <format>
        @type json
        time_format %Y-%m-%d %H:%M:%S.%L
        timezone +09:00
    </format>
</match>
```

</details>

<br>

<details><summary> :closed_book: Quiz 5. 컨테이너를 기동하고, 'docker run' 명령을 통해서 컨테이너 이름을 'quiz5-container'로 지정하고 'table_name' 항목으로 출력 되도록 컨테이너를 실행해 보세요</summary>


>  `docker-compose logs -f fluentd` 실행 결과에서 아래와 같이 메시지가 출력되면 성공입니다

```bash
fluentd  | 2022-07-24 05:29:17.802640130 +0000 docker.quiz5-container: {"container_id":"f996791c6897cd00d72376ae24d2052936b0964b65f8edc266a17b51131ba02c","container_name":"/quiz5-container","source":"stdout","log":"{\"message\":\"send message with name\"}","message":"send message with name","table_name":"quiz5-container"}
```

> 아래와 유사하게 `send_http.sh` 파일을 수정하시면 됩니다

```bash
docker run --rm --log-driver=fluentd --name 'quiz5-container' --log-opt tag=docker.{{.Name}} ubuntu echo '{"message":"send message with name"}'
```

</details>

<br>

[목차로 돌아가기](#2일차-스트림-데이터-수집)
