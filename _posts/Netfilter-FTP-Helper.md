---
published: true
date:
  'Sat Nov 12 2005 22:41:34 GMT+0800 (China Standard Time)': null
tags:
  - Linux
  - Netfilter
  - FTP
title: Netfilter FTP Helper源码分析
---

由于nat表需要ip conntrack模块支持，因此在针对FTP数据连接的nat动作也需要conntrack支持。主要流程如下：

netfilter hook将FTP控制连接加入到conntrack pool中，每一个在该conntrack上的数据报会被送至help函数（ip_conntrack_ftp.c），该函数由ip_conntrack_helper_unregister()注册成为该conntrack的helper，探测将会出现的期待连接信息，并注册入期待结构。当该helper函数注册的期待连接到达时（在这里就是FTP数据连接），该连接被注册为控制连接的RELATE conntrack。并且生成数据连接的第一个数据包，会被交给nat ftp helper 函数（ip_nat_ftp.c），该函数将该数据包payload中的地址信息（如：PORT、PASV命令中的ip地址）执行NAT。接下来该RELATE conntrack的第一个数据包会流入nat ftp expect函数（ip_nat_ftp.c）。由该函数执行NAT，执行过程和一般的NAT target差不多。

**接下来就分析这些helper，expect函数**

``` C
// file: ip_conntrack_ftp.c
// function: help

/*  每个控制连接的数据包会传入到该函数中。该函数探测该包是否包含FTP PASV/PORT命令。如果包含这些命令，则从命令中取得期待的数据连接的信息，ip及port。然后将该信息填入tuple/mask，注册到该conntrack期待的连接结构中。*/

static int help(const struct iphdr *iph, size_t len,
              struct ip_conntrack *ct,             /* 当前控制连接conntrack结构 */
              enum ip_conntrack_info ctinfo)  /* 控制连接状态信息 */
{
       struct tcphdr *tcph = (void *)iph + iph->ihl * 4;
       const char *data = (const char *)tcph + tcph->doff * 4;
       unsigned int tcplen = len – iph->ihl * 4;
       unsigned int datalen = tcplen – tcph->doff * 4;
       u_int32_t old_seq_aft_nl;
       int old_seq_aft_nl_set;
       u_int32_t array[6] = { 0 };    /* 存放payload及期待的ip,port */
       int dir = CTINFO2DIR(ctinfo);     /* ORIGIAN or REPLY */
       unsigned int matchlen, matchoff;
       struct ip_ct_ftp_master *ct_ftp_info = &ct->help.ct_ftp_info; /* 下一个期待的sequence */
        /*    声明一个ip_conntrack_expect期待结构，该结构稍后会被复制到内核kmalloc分配的空间里。该help函数的一个主要目的是，将获取的信息填充该结构，并把这个结构放入控制连接的conntrack中  */
       struct ip_conntrack_expect expect, *exp = &expect;
       struct ip_ct_ftp_expect *exp_ftp_info = &exp->help.exp_ftp_info;
       unsigned int i;

       int found = 0;

       /* FTP的PORT/PASV都应该在ESTABLESHED状态下出现 */

       if (ctinfo != IP_CT_ESTABLISHED
           && ctinfo != IP_CT_ESTABLISHED+IP_CT_IS_REPLY) {
              return NF_ACCEPT;
       }

       /* 检查TCP包头的完整性 */
       if (tcplen < sizeof(struct tcphdr) || tcplen < tcph->doff*4) {
              return NF_ACCEPT;
       }

       /* 检查tcp校验和 */
       if (tcp_v4_check(tcph, tcplen, iph->saddr, iph->daddr,
                      csum_partial((char *)tcph, tcplen, 0))) {
              return NF_ACCEPT;
       }

       LOCK_BH(&ip_ftp_lock);

       /* 获取期待的sequence */
       old_seq_aft_nl_set = ct_ftp_info->seq_aft_nl_set[dir];
       old_seq_aft_nl = ct_ftp_info->seq_aft_nl[dir];

       if ((datalen > 0) && (data[datalen-1] == ‘\n’)) {
              if (!old_seq_aft_nl_set || after(ntohl(tcph->seq) + datalen, old_seq_aft_nl)) {
                     /* 更新期望的下一个seqence */
                     ct_ftp_info->seq_aft_nl[dir] = ntohl(tcph->seq) + datalen;
                     ct_ftp_info->seq_aft_nl_set[dir] = 1;
              }
       }

       UNLOCK_BH(&ip_ftp_lock);

       if(!old_seq_aft_nl_set || (ntohl(tcph->seq) != old_seq_aft_nl)) {
              /* 该数据包并不是期待的那个，直接ACCEPT */
              return NF_ACCEPT;
       }
       /*  将该conntrack中tuple结构原地址取出，然后传递给find_pattern()，比较匹配。*/
       array[0] = (ntohl(ct->tuplehash[dir].tuple.src.ip) >> 24) & 0xFF;
       array[1] = (ntohl(ct->tuplehash[dir].tuple.src.ip) >> 16) & 0xFF;
       array[2] = (ntohl(ct->tuplehash[dir].tuple.src.ip) >> 8) & 0xFF;
       array[3] = ntohl(ct->tuplehash[dir].tuple.src.ip) & 0xFF;

       /*    开始遍历search结构。该结构包含一下信息:
              enum ip_conntrack_dir dir;     数据流方向
              const char *pattern;    匹配字符串
              size_t plen;           匹配长度
              char skip;             命令与参数之间需跳过的的字符
              char term;            分割符，如‘，’ip , port
              enum ip_ct_ftp_type ftptype;     FTP传输类型（PORT/PASV）
              int (*getnum)(const char *, size_t, u_int32_t[], char); 匹配函数
       */

       for (i = 0; i < sizeof(search) / sizeof(search[0]); i++) {
              if (search[i].dir != dir) continue;
              /*************************************
              *     匹配 “PORT” 或“227”（PASV的回应，
              *     包含server被动打开的port）
              *     find_pattern返回匹配的数据包中的 ip地址port信息
              *     存放在arrary数组中。同时返回匹配长度和偏移量。
              */
              found = find_pattern(data, datalen, search[i].pattern,
                                 search[i].plen, search[i].skip,
                                 search[i].term, &matchoff, &matchlen,
                                 array, search[i].getnum);
              if (found) break;
       }

       if (found == -1) {
             if (net_ratelimit())
              return NF_DROP;
       } else if (found == 0) /* No match */
              return NF_ACCEPT;
       /* 找到匹配 */
       memset(&expect, 0, sizeof(expect)); /* 清空expect结构，准备填充 */

       /* Update the ftp info */
       LOCK_BH(&ip_ftp_lock);

       if (htonl((array[0] << 24) | (array[1] << 16) | (array[2] << 8) | array[3])
           == ct->tuplehash[dir].tuple.src.ip) {
            /*    从数据包中获得的ip地址必须与tuple中源地址相同。
                     否则ACCEPT。防止ip欺骗 */
              exp->seq = ntohl(tcph->seq) + matchoff;  /* 期望的sequence = 匹配处*/
              exp_ftp_info->len = matchlen;      /* 匹配长度 */
              exp_ftp_info->ftptype = search[i].ftptype; /* PASV or PORT */
              exp_ftp_info->port = array[4] << 8 | array[5]; /* 期待的端口 */
       } else {
             if (!loose) goto out;
       }

       /*    填充期待的tuple/mask。
       *     tuple结构如下 {srcip, srcport, dstip, dstport, protonum}
       *     mask结构一样。{1, 0, 1, 1, 1}
       *     所以对tuple的匹配会忽略源端口。*/
       exp->tuple = ((struct ip_conntrack_tuple)
              { { ct->tuplehash[!dir].tuple.src.ip, /* 该方向上数据包目的地址 */
                  { 0 } },
                { htonl((array[0] << 24) | (array[1] << 16)
                       | (array[2] << 8) | array[3]),
                  { .tcp = { htons(array[4] << 8 | array[5]) } },
                  IPPROTO_TCP }});
       exp->mask = ((struct ip_conntrack_tuple)
              { { 0xFFFFFFFF, { 0 } },
                { 0xFFFFFFFF, { .tcp = { 0xFFFF } }, 0xFFFF }});

       exp->expectfn = NULL;
       /*    注册到expect related中当该期待的连接到达时（通过对数据包的tuple/mask匹配），ip conntrack就将其加入到ct的RELATED连接中。*/
       ip_conntrack_expect_related(ct, &expect);

 out:
       UNLOCK_BH(&ip_ftp_lock);
       return NF_ACCEPT;

}
```

**Nat 在conntrack注册了一个期待连接信息后，调用help直接修改那个引起期待的数据包。将PORT及227(PASV回应)中的ip地址及port做NAT。**

``` C
// file: ip_nat_ftp.c
// function: help

static unsigned int help(struct ip_conntrack *ct,
                      struct ip_conntrack_expect *exp,    /* 由conntrack help创建的expect */
                      struct ip_nat_info *info,
                      enum ip_conntrack_info ctinfo,
                      unsigned int hooknum,
                      struct sk_buff **pskb)
{
       struct iphdr *iph = (*pskb)->nh.iph;
       struct tcphdr *tcph = (void *)iph + iph->ihl*4;
       unsigned int datalen;
       int dir;
       struct ip_ct_ftp_expect *ct_ftp_info;

       if (!exp)  /* 没有expect连接，segment error！*/
              DEBUGP("ip_nat_ftp: no exp!!");

       /*    ct_ftp_info 结构包括ftp的数据传输类型，数据端口
        *     该结构再ip conntrack ftp help中创建
        */
       ct_ftp_info = &exp->help.exp_ftp_info;

       /* Only mangle things once: original direction in POST_ROUTING
          and reply direction on PRE_ROUTING. */
       dir = CTINFO2DIR(ctinfo);

       /* 只在两个情况下做help NAT */
       if (!((hooknum == NF_IP_POST_ROUTING && dir == IP_CT_DIR_ORIGINAL)
             || (hooknum == NF_IP_PRE_ROUTING && dir == IP_CT_DIR_REPLY))) {
              DEBUGP("nat_ftp: Not touching dir %s at hook %s\n",
                     dir == IP_CT_DIR_ORIGINAL ? "ORIG" : "REPLY",
                     hooknum == NF_IP_POST_ROUTING ? "POSTROUTING"
                     : hooknum == NF_IP_PRE_ROUTING ? "PREROUTING"
                     : hooknum == NF_IP_LOCAL_OUT ? "OUTPUT" : "???");
              return NF_ACCEPT;
       }

       datalen = (*pskb)->len – iph->ihl * 4 – tcph->doff * 4;
       LOCK_BH(&ip_ftp_lock);

       /* If it’s in the right range… */
       if (between(exp->seq + ct_ftp_info->len,     /* len 就是conntrack help 中的matchlen */
                  ntohl(tcph->seq),
                  ntohl(tcph->seq) + datalen)) {
               /*    需要修改的信息在该tcp payload中
				*     调用ftp_data_fixup修改FTP命令中的数据ip端口
				*     信息。
				*     ct_ftp_info结构包含：
				*            1) 匹配串中ip地址长度
				*            2) ftp数据传输类型
				*            3) 被动打开的tcp数据端口
				*     ct：当前conntrack结构，包含了conntrack的源、目的地址
				*     pskb：包缓存结构
				*     ctinfo：conntrack状态信息
				*     exp：期待连接结构，包含期待的字符串的起始sequence值，
				*            该值可以用来定位修改位置。
				*/

              if (!ftp_data_fixup(ct_ftp_info, ct, pskb, ctinfo, exp)) {
                     UNLOCK_BH(&ip_ftp_lock);
                     return NF_DROP;
              }
       } else {
              /* Half a match?  This means a partial retransmisison.
                 It’s a cracker being funky. */
              if (net_ratelimit()) {
                     printk("FTP_NAT: partial packet %u/%u in %u/%u\n",
                            exp->seq, ct_ftp_info->len,
                            ntohl(tcph->seq),
                            ntohl(tcph->seq) + datalen);
              }
              UNLOCK_BH(&ip_ftp_lock);
              return NF_DROP;
       }

       UNLOCK_BH(&ip_ftp_lock);
       return NF_ACCEPT;
}
```

**当第一个期待连接的数据包到来时，下面的expect函数被调用。该函数的作用和nat table中的target差不多。都是向nat core注册一个nat的ip_nat_multi_range结构。该结构就是nat所需的转换地址。**

``` C
// file: ip_nat_ftp.c
// function: ftp_nat_expect

static unsigned int
ftp_nat_expected(struct sk_buff **pskb,
               unsigned int hooknum,
               struct ip_conntrack *ct,
               struct ip_nat_info *info)
{
       struct ip_nat_multi_range mr;
       u_int32_t newdstip, newsrcip, newip;
       struct ip_ct_ftp_expect *exp_ftp_info;
       struct ip_conntrack *master = master_ct(ct); /* 主conntrack */

       IP_NF_ASSERT(info);
       IP_NF_ASSERT(master);
       IP_NF_ASSERT(!(info->initialized & (1<<HOOK2MANIP(hooknum))));

       DEBUGP("nat_expected: We have a connection!\n");
       exp_ftp_info = &ct->master->help.exp_ftp_info;

       LOCK_BH(&ip_ftp_lock);

       /*    根据主conntrack中ftp的数据传输类型类型
		*     来获取特定的需要转换的源、目的ip地址
 		*/

       if (exp_ftp_info->ftptype == IP_CT_FTP_PORT
           || exp_ftp_info->ftptype == IP_CT_FTP_EPRT) {
              /* PORT command: make connection go to the client. */
              /*    数据传输类型为PORT，那么第一个数据连接包是由server主动连接
              *     到client，所以将DIR_ORIGINAL的conntrack的源，目的地址交换。
              *     作为nat的目的，源地址。
              */
              newdstip = master->tuplehash[IP_CT_DIR_ORIGINAL].tuple.src.ip;
              newsrcip = master->tuplehash[IP_CT_DIR_ORIGINAL].tuple.dst.ip;
              DEBUGP("nat_expected: PORT cmd. %u.%u.%u.%u->%u.%u.%u.%u\n",
                     NIPQUAD(newsrcip), NIPQUAD(newdstip));
       } else {
              /* PASV command: make the connection go to the server */
              /*    数据传输类型为PASV，那么第一个数据连接包是由client主动连接
              *     到server，所以将DIR_REPLY的conntrack（server发出的数据流）
			  *     的源，目的地址交换。作为nat的目的，源地址。
              */

              newdstip = master->tuplehash[IP_CT_DIR_REPLY].tuple.src.ip;
              newsrcip = master->tuplehash[IP_CT_DIR_REPLY].tuple.dst.ip;
              DEBUGP("nat_expected: PASV cmd. %u.%u.%u.%u->%u.%u.%u.%u\n",
                     NIPQUAD(newsrcip), NIPQUAD(newdstip));
       }

       UNLOCK_BH(&ip_ftp_lock);
       /*    根据hook位置选择nat的ip地址
		*     如果是SNAT， newip = newsrcip
		*     是DNAT，newip = newdstip
		*/

       if (HOOK2MANIP(hooknum) == IP_NAT_MANIP_SRC)
              newip = newsrcip;
       else
              newip = newdstip;

       DEBUGP("nat_expected: IP to %u.%u.%u.%u\n", NIPQUAD(newip));
       mr.rangesize = 1; /* 该nat不做随机的选择 */
       /* We don’t want to manip the per-protocol, just the IPs… */
       mr.range[0].flags = IP_NAT_RANGE_MAP_IPS;
       mr.range[0].min_ip = mr.range[0].max_ip = newip; /* nat 到newip */
       
       /* … unless we’re doing a MANIP_DST, in which case, make
          sure we map to the correct port */
       if (HOOK2MANIP(hooknum) == IP_NAT_MANIP_DST) {
              /*    不要忘了还有port转换。exp_ftp_info->port就是由
				conntrack help 中查找tcp payload获得*/
              mr.range[0].flags |= IP_NAT_RANGE_PROTO_SPECIFIED;
              mr.range[0].min = mr.range[0].max
                     = ((union ip_conntrack_manip_proto)
                            { .tcp = { htons(exp_ftp_info->port) } });
       }

       return ip_nat_setup_info(ct, &mr, hooknum); /* 注册入nat core中 */
}
```
