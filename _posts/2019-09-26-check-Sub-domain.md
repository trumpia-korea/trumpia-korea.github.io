---
title: "Post: Check site Sub-domains"
categories:
  - security
tags:
  - Infra
  - Security

author: joseph
---

# Sub-domain 검사하기

> 작성일: 2019-04-09
>
> 작성자: 유요셉
>
> > 해당 문서는 Kali Linux 2019.1 Version을 기준으로 작성되었습니다.



## Sublist3r

> Sublist3r은 python으로 작성된 script로, Kali Linux(2019.01 Version 기준)에는 포함되어 있지 않다.



### Download Sublist3r

*Sublist3r*은 GitHub을 통해 code를 제공한다. Google 검색 또는 [해당 URL](https://github.com/aboul3la/Sublist3r)을 통해 접속할 수 있다.

아래 명령으로 code를 다운로드한다.

```bash
git clone https://github.com/aboul3la/Sublist3r.git
```



### Recommended Python Version

Sublist3r currently supports **Python 2** and **Python 3**.



### Dependencies

다운로드한 code의 directory에서 requirements.txt를 확인할 수 있다.

Kali Linux에는 기본적으로 설치된 python module이지만, 설치가 되어있지 않다면 다음 명령으로 설치할 수 있다.

```bash
sudo pip install -r requirements.txt
```



### Sublist3r Command

#### Help

```bash
python sublist3r.py -h
```

#### Options

| Short Form | Long Form | Description |
| ---------- | --------- |------------ |
| -d | --domain | Domain name to enumerate subdomains of |
| -b | --bruteforce | Enable the subbrute bruteforce module |
| -p | --ports | Scan the found subdomains against specific tcp ports |
| -v | --verbose | Enable the verbose mode and display results in realtime |
| -t | --threads | Number of threads to use for subbrute bruteforce |
| -e | --engines | Specify a comma-separated list of search engines |
| -o | --output | Save the results to text file |
| -h | --help | show the help message and exit |



### Execute Sublist3r

#### Basic Search

```shell
python sublist3r.py -d example.com
```

#### Specific Port Search

```shell
python sublist3r.py -d example.com -p 80,443
```

#### Using Specific Engine Search

```shell
python sublist3r.py -e google,yahoo,virustotal -d example.com
```



***



## fierce

> fierce는 Kali Linux에 포함된 domain search tool이다.



### fierce Command

#### Help

```shell
fierce -h
```

#### Options

주요한 옵션은 다음과 같다.

| Option   | Description |
| -------- | ----------- |
| -connect | Attempt to make http connections to any non RFC1918(public) addresses. |
| -delay | The number of seconds to wait between lookups. |
| **-dns** | The domain you would like scanned. |
| -dnsserver | Use a particular DNS server for reverse lookups. |
| -dnsfile | Use DNS servers provided by a file (one per line) for reverse lookups<br />(brute force). |
| -file | A file you would like to output to be logged to. |
| -fileoutput | When combined with -connect this will output everything <br />the webserver sends back, not just the HTTP headers. |
| -range | Scan an internal IP range. <br />(must be combined with -dnsserver) |



### Execute fierce

#### Basic Search

```shell
fierce -dns example.com
```



***



## dnsmap

> fierce는 Kali Linux에 포함된 domain search tool이다.



### dnsmap Command

#### Help

```shell
dnsmap
```

> 별도의 help option이 없다.

단순히 tool 이름만 입력하면 사용법을 확인할 수 있다.

#### Options

주요한 옵션은 다음과 같다.

| Option   | Description |
| -------- | ----------- |
| -w | wordlist-file: Input dictionary file. |
| -r | regular-results-file: Save raw result. |
| -c | csv-results-file: Save CSV formatted result. |
| -d | delay-millisecs |
| -i | ips-to-ignore <br />(useful if you're obtaining false positives) |



### Execute dnsmap

#### Basic Search

```shell
dnsmap example.com
```

#### Using Provided Dictionary File Search

```shell
dnsmap example.com -w /usr/share/wordlists/dnsmap.txt
```

#### Save Search Result

```shell
dnsmap example.com -r result.txt
```



***

