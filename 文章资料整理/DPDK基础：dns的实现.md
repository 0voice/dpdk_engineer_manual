# DPDK基础：dns的实现

之前，我们实现了用dpdk处理udp协议数据。这里，我们在udp基础上实现dns，其实是借用了开源框架，懒得自己去实现解析了。

## 1.再谈arp

这里我们没有使用自己实现的arp，而是把arp包交给kni处理。为什么辛辛苦苦实现arp，最后却不用呢？因为我们没有实现记录arp的功能，网卡在接收到arp包后，给发送方回了arp包，但是自己却没有处理这个包。arp表上没有相应ip与mac的映射，那就发送不了数据，kni会帮我们添加arp表项。
使用kni，又会有一个问题，kni默认是不进行回环的，也就是说，接受到了数据不会回发。设置kni回发，有三种方法。
法一：修改/sys/devices/virtual/net/%s/carrier

```c++
echo 1 > /sys/devices/virtual/net/%s/carrier

```

法二：在dpdk目录下的x86_64-native-linux-gcc/kmod路径下

```c++
insmod rte_kni.ko carrier=on # 注意root

```

法三：代码中调用rte_kni_update_link()函数，该函数的核心功能就是将1写入`/sys/devices/virtual/net/%s/carrier`文件。
在代码中，我们使用第三种方法。

```c++
while (1) {
		rte_delay_ms(500);
			
		memset(&link, 0, sizeof(link));
		rte_eth_link_get_nowait(g_dpdkPortId, &link);
			
		prev = rte_kni_update_link(kni, link.link_status);
		// echo 1 > /sys/devices/virtual/net/vEth0/carrier
		log_link_state(kni, prev, &link);
}

```

由于`/sys/devices/virtual/net/%s/carrier`值是公共资源，为了避免被其他进程修改，这里开了一个线程，每隔500ms检测并修改值为1。

```c++
//main函数中
pthread_t kni_link_tid;
int ret = rte_ctrl_thread_create(&kni_link_tid,
				 "KNI link status check", NULL,
				 monitor_all_ports_link_status, NULL);
if (ret < 0)
	rte_exit(EXIT_FAILURE, "Could not create link status thread!\n");

```

## 2.dns工具

介绍两个dns工具。

### 2.1dig

dig（domain information group）是常用的域名查询工具，可以从DNS域名服务器查询主机地址信息，获取到详细的域名信息。这个命令是Bind的一部分，本身并没有在Windows和Linux系统中集成，所以如果我们想要使用该命令就需要先下载相应的软件包。

```c++
apt-get install dnsutils

```

### 2.2dns数据类型

给张表

![在这里插入图片描述](https://img-blog.csdnimg.cn/03f1021efc7c4a03bf0b2d53b4ae080b.jpg?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5ZSP5ZmX,size_20,color_FFFFFF,t_70,g_se,x_16)

### 2.3查询命令

查询记录的类型

```c++
dig www.baidu.com A     # 查询A记录，如果域名后面不加任何参数，默认查询A记录
dig www.baidu.com MX    # 查询MX记录
dig www.baidu.com CNAME # 查询CNAME记录
dig www.baidu.com NS	# 查询NS记录
dig www.baidu.com ANY	# 查询上面所有的记录

```

```c++
dig www.baidu.com A +short	# 查询A记录并显示简要的返回的结果
dig www.baidu.com A +multiline	# 查询A记录并显示详细的返回结果

```

查询指定dns服务器

```c++
dig @192.168.0.2 www.baidu.com # 表示从192.168.0.2这个IP服务器对www.baidu.com进行A记录查询

```

若不指定DNS服务器，dig会依次使用/etc/resolv.conf里的地址作为DNS服务器。

### 2.4查询结果

以查询百度域名为例

```c++
dig www.baidu.com

```

下面是输出的结果，注释是后加的，为了方便解释。

```c++
# 这部分输出版本信息(version 9.10.3)和全局的设置选项
; <<>> DiG 9.10.3-P4-Ubuntu <<>> www.baidu.com
;; global options: +cmd

# 这部分输出从DNS返回的技术信息，比较重要的是 status，如果 status 的值为 NOERROR ，则说明本次查询成功
# 这段信息可以用选项 [no]comments 来控制是否显示，但是小心禁止掉comments也可能关闭一些其它的选项
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5575
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
# 查询字段，显示了我们要查询的域名以及查询的服务，A表示的是A记录查询，即主机查询
;; QUESTION SECTION:
;www.baidu.com.			IN	A

# 这一部分是DNS服务器的答复
# 显示www.baidu.com.有一个CNAME记录，以及www.a.shifen.com.有两个A记录
# 415、146是TTL值，表示缓存时间，即415s内不用重新查询
;; ANSWER SECTION:
www.baidu.com.		415	IN	CNAME	www.a.shifen.com.
www.a.shifen.com.	146	IN	A	14.215.177.38
www.a.shifen.com.	146	IN	A	14.215.177.39

# 这部分显示了请求所花的时间，dns服务器和端口，当前时间，以及查询信息的大小
;; Query time: 2 msec
;; SERVER: 192.168.2.1#53(192.168.2.1)
;; WHEN: Sat Sep 03 01:39:18 PDT 2022
;; MSG SIZE  rcvd: 104

```

当然，dig还有一些其他的命令操作，但是在这里介绍这些已经够用了。
再介绍另一个工具

## 3.dnsperf

DNSPerf（DNS Performance）是一款dns压力测试工具，从全世界超过两百个城市节点来检测各个 DNS 速度、反应时间及上线率（Uptime），除此之外，DNSPerf 还有针对一般使用者会用到的开放式 DNS 解析服务（Public DNS）进行监测记录。
命令安装

```c++
apt-get install dnsperf

```

也可以源码安装，github地址https://github.com/cobblau/dnsperf

### 3.1参数

dnsperf 支持下面的这些命令行参数

```c++
-s    用来指定DNS服务器的IP地址，默认值是127.0.0.1
-p    用来指定DNS服务器的端口，默认值是53
-d    用来指定DNS消息的内容文件，该文件中包含要探测的域名和资源记录类型
-t    用来指定每个请求的超时时间，默认值是3000ms
-Q    用来指定本次压测的最大请求数，默认值是1000
-c    用来指定并发探测数，默认值是100. dnsperf会从-d指定的文件中随机选取100个座位探测域名来发送DNS请求
-l    用来指定本次压测的时间，默认值是无穷大
-e    本选项通过EDNS0，在OPT资源记录中运用edns-client-subnet来指定真实的client ip
-i    用来指定前后探测的时间间隔，因为dnsperf是一个压测工具，所以本选项目前还不支持
-P    指定用哪个传输层协议发送DNS请求，udp或者tcp。默认值是udp
-f    指定用什么地址类型发送DNS请求，inet或者inet6。默认值是inet
-v    除了标准的输出外，还输出每个相应码的个数
-h    打印帮助

```

### 3.2使用命令

在使用之前，还需要写一个文档

```c++
# vi testfile，在testfile中
www.baidu.com A

```

执行命令

```c++
./src/dnsperf -d tsetfile -s 192.168.0.2 -c10000 -Q10000 -l60

```

执行结果

```c++
DNS Performance Testing Tool
 
[Status] Processing query data
[Status] Sending queries to 127.0.0.1:53
time up
[Status]DNS Query Performance Testing Finish
[Result]Quries sent:        35650
[Result]Quries completed:   35578
[Result]Complete percentage:    99.80%
 
[Result]Elapsed time(s):    1.00000
 
[Result]Queries Per Second: 35650.0000

```

## 4.实现dns

这里我们使用simpleDNS框架来实现
先从git下载
https://github.com/mwarning/SimpleDNS.git
然后使用其中的源码，其实是直接拷贝……

```c++
#ifndef __DPDK_DNS_H__
#define __DPDK_DNS_H__
/* Response Type */
enum {
  Ok_ResponseType = 0,
  FormatError_ResponseType = 1,
  ServerFailure_ResponseType = 2,
  NameError_ResponseType = 3,
  NotImplemented_ResponseType = 4,
  Refused_ResponseType = 5
};

/* Resource Record Types */
enum {
  A_Resource_RecordType = 1,
  NS_Resource_RecordType = 2,
  CNAME_Resource_RecordType = 5,
  SOA_Resource_RecordType = 6,
  PTR_Resource_RecordType = 12,
  MX_Resource_RecordType = 15,
  TXT_Resource_RecordType = 16,
  AAAA_Resource_RecordType = 28,
  SRV_Resource_RecordType = 33
};

/* Operation Code */
enum {
  QUERY_OperationCode = 0, /* standard query */
  IQUERY_OperationCode = 1, /* inverse query */
  STATUS_OperationCode = 2, /* server status request */
  NOTIFY_OperationCode = 4, /* request zone transfer */
  UPDATE_OperationCode = 5 /* change resource records */
};

/* Response Code */
enum {
  NoError_ResponseCode = 0,
  FormatError_ResponseCode = 1,
  ServerFailure_ResponseCode = 2,
  NameError_ResponseCode = 3
};

/* Query Type */
enum {
  IXFR_QueryType = 251,
  AXFR_QueryType = 252,
  MAILB_QueryType = 253,
  MAILA_QueryType = 254,
  STAR_QueryType = 255
};

/*
* Types.
*/

/* Question Section */
struct Question {
  char *qName;
  uint16_t qType;
  uint16_t qClass;
  struct Question *next; // for linked list
};

/* Data part of a Resource Record */
union ResourceData {
  struct {
    uint8_t txt_data_len;
    char *txt_data;
  } txt_record;
  struct {
    uint8_t addr[4];
  } a_record;
  struct {
    uint8_t addr[16];
  } aaaa_record;
};

/* Resource Record Section */
struct ResourceRecord {
  char *name;
  uint16_t type;
  uint16_t class;
  uint32_t ttl;
  uint16_t rd_length;
  union ResourceData rd_data;
  struct ResourceRecord *next; // for linked list
};

struct Message {
  uint16_t id; /* Identifier */

  /* Flags */
  uint16_t qr; /* Query/Response Flag */
  uint16_t opcode; /* Operation Code */
  uint16_t aa; /* Authoritative Answer Flag */
  uint16_t tc; /* Truncation Flag */
  uint16_t rd; /* Recursion Desired */
  uint16_t ra; /* Recursion Available */
  uint16_t rcode; /* Response Code */

  uint16_t qdCount; /* Question Count */
  uint16_t anCount; /* Answer Record Count */
  uint16_t nsCount; /* Authority Record Count */
  uint16_t arCount; /* Additional Record Count */

  /* At least one question; questions are copied to the response 1:1 */
  struct Question *questions;

  /*
  * Resource records to be send back.
  * Every resource record can be in any of the following places.
  * But every place has a different semantic.
  */
  struct ResourceRecord *answers;
  struct ResourceRecord *authorities;
  struct ResourceRecord *additionals;
};


int decode_msg(struct Message *msg, const uint8_t *buffer, int size);

void resolve_query(struct Message *msg);

int encode_msg(struct Message *msg, uint8_t **buffer);

void free_questions(struct Question *qq);

void free_resource_records(struct ResourceRecord *rr);

void print_message(struct Message *msg);

#endif

```

再拷实现，注意要把内存操作malloc、free改为rte_前缀的，因为我们是使用dpdk实现的。

```c++
/*
* This software is licensed under the CC0.
*
* This is a _basic_ DNS Server for educational use.
* It does not prevent invalid packets from crashing
* the server.
*
* To test start the program and issue a DNS request:
*  dig @127.0.0.1 -p 9000 foo.bar.com 
*/


/*
* Masks and constants.
*/

static const uint32_t QR_MASK = 0x8000;
static const uint32_t OPCODE_MASK = 0x7800;
static const uint32_t AA_MASK = 0x0400;
static const uint32_t TC_MASK = 0x0200;
static const uint32_t RD_MASK = 0x0100;
static const uint32_t RA_MASK = 0x8000;
static const uint32_t RCODE_MASK = 0x000F;


int get_A_Record(uint8_t addr[4], const char domain_name[])
{
  if (strcmp("foo.bar.com", domain_name) == 0) {
    addr[0] = 192;
    addr[1] = 168;
    addr[2] = 232;
    addr[3] = 133;
    return 0;
  } else {
    return -1;
  }
}

int get_AAAA_Record(uint8_t addr[16], const char domain_name[])
{
  if (strcmp("foo.bar.com", domain_name) == 0) {
    addr[0] = 0xfe;
    addr[1] = 0x80;
    addr[2] = 0x00;
    addr[3] = 0x00;
    addr[4] = 0x00;
    addr[5] = 0x00;
    addr[6] = 0x00;
    addr[7] = 0x00;
    addr[8] = 0x00;
    addr[9] = 0x00;
    addr[10] = 0x00;
    addr[11] = 0x00;
    addr[12] = 0x00;
    addr[13] = 0x00;
    addr[14] = 0x00;
    addr[15] = 0x01;
    return 0;
  } else {
    return -1;
  }
}

int get_TXT_Record(char **addr, const char domain_name[])
{
  if (strcmp("foo.bar.com", domain_name) == 0) {
    *addr = "abcdefg";
    return 0;
  } else {
    return -1;
  }
}

/*
* Debugging functions.
*/

void print_hex(uint8_t *buf, size_t len)
{
  int i;
  printf("%zu bytes:\n", len);
  for (i = 0; i < len; ++i)
    printf("%02x ", buf[i]);
  printf("\n");
}

void print_resource_record(struct ResourceRecord *rr)
{
  int i;
  while (rr) {
    printf("  ResourceRecord { name '%s', type %u, class %u, ttl %u, rd_length %u, ",
      rr->name,
      rr->type,
      rr->class,
      rr->ttl,
      rr->rd_length
   );

    union ResourceData *rd = &rr->rd_data;
    switch (rr->type) {
      case A_Resource_RecordType:
        printf("Address Resource Record { address ");

        for(i = 0; i < 4; ++i)
          printf("%s%u", (i ? "." : ""), rd->a_record.addr[i]);

        printf(" }");
        break;
      case AAAA_Resource_RecordType:
        printf("AAAA Resource Record { address ");

        for(i = 0; i < 16; ++i)
          printf("%s%02x", (i ? ":" : ""), rd->aaaa_record.addr[i]);

        printf(" }");
        break;
      case TXT_Resource_RecordType:
        printf("Text Resource Record { txt_data '%s' }",
          rd->txt_record.txt_data
       );
        break;
      default:
        printf("Unknown Resource Record { ??? }");
    }
    printf("}\n");
    rr = rr->next;
  }
}

void print_message(struct Message *msg)
{
  struct Question *q;

  printf("QUERY { ID: %02x", msg->id);
  printf(". FIELDS: [ QR: %u, OpCode: %u ]", msg->qr, msg->opcode);
  printf(", QDcount: %u", msg->qdCount);
  printf(", ANcount: %u", msg->anCount);
  printf(", NScount: %u", msg->nsCount);
  printf(", ARcount: %u,\n", msg->arCount);

  q = msg->questions;
  while (q) {
    printf("  Question { qName '%s', qType %u, qClass %u }\n",
      q->qName,
      q->qType,
      q->qClass
    );
    q = q->next;
  }

  print_resource_record(msg->answers);
  print_resource_record(msg->authorities);
  print_resource_record(msg->additionals);

  printf("}\n");
}


/*
* Basic memory operations.
*/

size_t get16bits(const uint8_t **buffer)
{
  uint16_t value;

  memcpy(&value, *buffer, 2);
  *buffer += 2;

  return ntohs(value);
}

void put8bits(uint8_t **buffer, uint8_t value)
{
  memcpy(*buffer, &value, 1);
  *buffer += 1;
}

void put16bits(uint8_t **buffer, uint16_t value)
{
  value = htons(value);
  memcpy(*buffer, &value, 2);
  *buffer += 2;
}

void put32bits(uint8_t **buffer, uint32_t value)
{
  value = htonl(value);
  memcpy(*buffer, &value, 4);
  *buffer += 4;
}


/*
* Deconding/Encoding functions.
*/

// 3foo3bar3com0 => foo.bar.com (No full validation is done!)
char *decode_domain_name(const uint8_t **buf, size_t len)
{
  char domain[256];
  for (int i = 1; i < MIN(256, len); i += 1) {
    uint8_t c = (*buf)[i];
    if (c == 0) {
      domain[i - 1] = 0;
      *buf += i + 1;
      return strdup(domain);
    } else if (c <= 63) {
      domain[i - 1] = '.';
    } else {
      domain[i - 1] = c;
    }
  }

  return NULL;
}

// foo.bar.com => 3foo3bar3com0
void encode_domain_name(uint8_t **buffer, const char *domain)
{
  uint8_t *buf = *buffer;
  const char *beg = domain;
  const char *pos;
  int len = 0;
  int i = 0;

  while ((pos = strchr(beg, '.'))) {
    len = pos - beg;
    buf[i] = len;
    i += 1;
    memcpy(buf+i, beg, len);
    i += len;

    beg = pos + 1;
  }

  len = strlen(domain) - (beg - domain);

  buf[i] = len;
  i += 1;

  memcpy(buf + i, beg, len);
  i += len;

  buf[i] = 0;
  i += 1;

  *buffer += i;
}


void decode_header(struct Message *msg, const uint8_t **buffer)
{
  msg->id = get16bits(buffer);

  uint32_t fields = get16bits(buffer);
  msg->qr = (fields & QR_MASK) >> 15;
  msg->opcode = (fields & OPCODE_MASK) >> 11;
  msg->aa = (fields & AA_MASK) >> 10;
  msg->tc = (fields & TC_MASK) >> 9;
  msg->rd = (fields & RD_MASK) >> 8;
  msg->ra = (fields & RA_MASK) >> 7;
  msg->rcode = (fields & RCODE_MASK) >> 0;

  msg->qdCount = get16bits(buffer);
  msg->anCount = get16bits(buffer);
  msg->nsCount = get16bits(buffer);
  msg->arCount = get16bits(buffer);
}

void encode_header(struct Message *msg, uint8_t **buffer)
{
  put16bits(buffer, msg->id);

  int fields = 0;
  fields |= (msg->qr << 15) & QR_MASK;
  fields |= (msg->rcode << 0) & RCODE_MASK;
  // TODO: insert the rest of the fields
  put16bits(buffer, fields);

  put16bits(buffer, msg->qdCount);
  put16bits(buffer, msg->anCount);
  put16bits(buffer, msg->nsCount);
  put16bits(buffer, msg->arCount);
}

int decode_msg(struct Message *msg, const uint8_t *buffer, int size)
{
  int i;

  decode_header(msg, &buffer);

  if (msg->anCount != 0 || msg->nsCount != 0) {
    printf("Only questions expected!\n");
    return -1;
  }

  // parse questions
  uint32_t qcount = msg->qdCount;
  for (i = 0; i < qcount; ++i) {
    struct Question *q = rte_malloc("Question", sizeof(struct Question), 0);

    q->qName = decode_domain_name(&buffer, size);
    q->qType = get16bits(&buffer);
    q->qClass = get16bits(&buffer);

    if (q->qName == NULL) {
      printf("Failed to decode domain name!\n");
      return -1;
    }

    // prepend question to questions list
    q->next = msg->questions;
    msg->questions = q;
  }

  // We do not expect any resource records to parse here.

  return 0;
}

// For every question in the message add a appropiate resource record
// in either section 'answers', 'authorities' or 'additionals'.
void resolve_query(struct Message *msg)
{
  struct ResourceRecord *beg;
  struct ResourceRecord *rr;
  struct Question *q;
  int rc;

  // leave most values intact for response
  msg->qr = 1; // this is a response
  msg->aa = 1; // this server is authoritative
  msg->ra = 0; // no recursion available
  msg->rcode = Ok_ResponseType;

  // should already be 0
  msg->anCount = 0;
  msg->nsCount = 0;
  msg->arCount = 0;

  // for every question append resource records
  q = msg->questions;
  while (q) {
    rr = rte_malloc("ResourceRecord", sizeof(struct ResourceRecord), 0); //malloc
    memset(rr, 0, sizeof(struct ResourceRecord));

    rr->name = strdup(q->qName);
    rr->type = q->qType;
    rr->class = q->qClass;
    rr->ttl = 60*60; // in seconds; 0 means no caching

    //printf("Query for '%s'\n", q->qName);

    // We only can only answer two question types so far
    // and the answer (resource records) will be all put
    // into the answers list.
    // This behavior is probably non-standard!
    switch (q->qType) {
      case A_Resource_RecordType:
        rr->rd_length = 4;
        rc = get_A_Record(rr->rd_data.a_record.addr, q->qName);
        if (rc < 0)
        {
          rte_free(rr->name);
          rte_free(rr);
          goto next;
        }
        break;
      case AAAA_Resource_RecordType:
        rr->rd_length = 16;
        rc = get_AAAA_Record(rr->rd_data.aaaa_record.addr, q->qName);
        if (rc < 0)
        {
          rte_free(rr->name);
          rte_free(rr);
          goto next;
        }
        break;
      case TXT_Resource_RecordType:
        rc = get_TXT_Record(&(rr->rd_data.txt_record.txt_data), q->qName);
        if (rc < 0) {
          rte_free(rr->name);
          rte_free(rr);
          goto next;
        }
        int txt_data_len = strlen(rr->rd_data.txt_record.txt_data);
        rr->rd_length = txt_data_len + 1;
        rr->rd_data.txt_record.txt_data_len = txt_data_len;
        break;
      /*
      case NS_Resource_RecordType:
      case CNAME_Resource_RecordType:
      case SOA_Resource_RecordType:
      case PTR_Resource_RecordType:
      case MX_Resource_RecordType:
      case TXT_Resource_RecordType:
      */
      default:
        rte_free(rr);
        msg->rcode = NotImplemented_ResponseType;
        printf("Cannot answer question of type %d.\n", q->qType);
        goto next;
    }

    msg->anCount++;

    // prepend resource record to answers list
    beg = msg->answers;
    msg->answers = rr;
    rr->next = beg;

    // jump here to omit question
    next:

    // process next question
    q = q->next;
  }
}

/* @return 0 upon failure, 1 upon success */
int encode_resource_records(struct ResourceRecord *rr, uint8_t **buffer)
{
  int i;
  while (rr) {
    // Answer questions by attaching resource sections.
    encode_domain_name(buffer, rr->name);
    put16bits(buffer, rr->type);
    put16bits(buffer, rr->class);
    put32bits(buffer, rr->ttl);
    put16bits(buffer, rr->rd_length);

    switch (rr->type) {
      case A_Resource_RecordType:
        for(i = 0; i < 4; ++i)
          put8bits(buffer, rr->rd_data.a_record.addr[i]);
        break;
      case AAAA_Resource_RecordType:
        for(i = 0; i < 16; ++i)
          put8bits(buffer, rr->rd_data.aaaa_record.addr[i]);
        break;
      case TXT_Resource_RecordType:
        put8bits(buffer, rr->rd_data.txt_record.txt_data_len);
        for(i = 0; i < rr->rd_data.txt_record.txt_data_len; i++)
          put8bits(buffer, rr->rd_data.txt_record.txt_data[i]);
        break;
      default:
        fprintf(stderr, "Unknown type %u. => Ignore resource record.\n", rr->type);
      return 1;
    }

    rr = rr->next;
  }

  return 0;
}

/* @return 0 upon failure, 1 upon success */
int encode_msg(struct Message *msg, uint8_t **buffer)
{
  struct Question *q;
  int rc;

  encode_header(msg, buffer);

  q = msg->questions;
  while (q) {
    encode_domain_name(buffer, q->qName);
    put16bits(buffer, q->qType);
    put16bits(buffer, q->qClass);

    q = q->next;
  }

  rc = 0;
  rc |= encode_resource_records(msg->answers, buffer);
  rc |= encode_resource_records(msg->authorities, buffer);
  rc |= encode_resource_records(msg->additionals, buffer);

  return rc;
}

void free_resource_records(struct ResourceRecord *rr)
{
  struct ResourceRecord *next;

  while (rr) {
    rte_free(rr->name);
    next = rr->next;
    rte_free(rr);
    rr = next;
  }
}

void free_questions(struct Question *qq)
{
  struct Question *next;

  while (qq) {
    rte_free(qq->qName);
    next = qq->next;
    rte_free(qq);
    qq = next;
  }
}

```

再在我们之前实现的代码中进行修改，改一下udp的实现

```c++
if (UDP_PORT == ntohs(udp_hdr->dst_port)) {
#if ENABLE_DNS

	g_src_ip = ip_hdr->dst_addr;
	g_dest_ip = ip_hdr->src_addr;

	g_src_port = ntohs(udp_hdr->dst_port);
	g_dest_port = ntohs(udp_hdr->src_port);

	rte_memcpy(g_dest_mac_addr, ehdr->s_addr.addr_bytes, RTE_ETHER_ADDR_LEN);

	// udp userdata
	uint16_t length = ntohs(udp_hdr->dgram_len);
	uint16_t nbytes = length - sizeof(struct rte_udp_hdr);

	uint8_t *data = (uint8_t*)(udp_hdr+1);
						

	// decode_msg

	free_questions(msg.questions);
	free_resource_records(msg.answers);
	free_resource_records(msg.authorities);
	free_resource_records(msg.additionals);
	memset(&msg, 0, sizeof(struct Message));
						
	decode_msg(&msg, data, nbytes);
						
	// resolve_query
	resolve_query(&msg); 

	// encode_msg
	uint8_t *p = data;				
	encode_msg(&msg, &p);

	// send_udp_pkt
	int len = p - data;
	do_send_udp(pktmbuf_pool, data, len);
}

```

其实就四步，解析数据、处理数据、编码数据、回发。
编译完成后运行代码，注意运行之前设置kni回发，运行后执行命令`ifconfig vEth0 192.168.0.120 hw 00:0c:29:85:2e:88 up`，然后就可以使用刚才介绍的dns测试工具进行测试了。







原文链接：https://blog.csdn.net/m0_65931372/article/details/126719463

原文作者:唏噗