---
published: true
date: 2006-03-01T13:00:00.000Z
tags:
  - Linux
  - Netfilter
  - TCP
title: 如何在kernel里发送RST包
---

kernel里发送RST包比较麻烦，首先你需要构造一个skb就够（很害怕吧！呵呵），再次要填充skb->data。当然就是填充ip、tcp头了。第一步，我们可以偷懒，用先前接收到的skb（RST肯定是在接收到SYN，或者主动异常中止链接，之前对方的包已收到）做为蓝本来构造RST的skb，或者干脆就拿这个skb来用。在有netfilter的kernel里，最典范的代码应该在ipt_REJECT target中。以下摘录，并分析它：

<!-- more -->

``` C

static void send_reset(struct sk_buff *oldskb, int local)
{
    struct sk_buff *nskb;
    struct tcphdr *otcph, *tcph;
    struct rtable *rt;
    unsigned int otcplen;
    u_int16_t tmp_port;
    u_int32_t tmp_addr;
    int needs_ack;
    int hh_len;

    /* IP header checks: fragment, too short. */
    if (oldskb->nh.iph->frag_off & htons(IP_OFFSET)
        || oldskb->len < (oldskb->nh.iph->ihl<<2) + sizeof(struct tcphdr))
        return;
    // 获取老的skb的tcp头及tcp头及数据长度
    otcph = (struct tcphdr *)((u_int32_t*)oldskb->nh.iph + oldskb->nh.iph->ihl);
    otcplen = oldskb->len – oldskb->nh.iph->ihl*4;

    /* No RST for RST. */
    if (otcph->rst) // 收到的是RST包，显然不能再回RST
        return;

    /* Check checksum. */
    if (tcp_v4_check(otcph, otcplen, oldskb->nh.iph->saddr,
             oldskb->nh.iph->daddr,
             csum_partial((char *)otcph, otcplen, 0)) != 0)
        return; // tcp checksum 检查

    if ((rt = route_reverse(oldskb, local)) == NULL)
        return; // 将老的skb中ip路由进行翻转，得到的路由表引用放入rt

    hh_len = (rt->u.dst.dev->hard_header_len + 15)&~15; // MAC头长度

    /* Copy skb (even if skb is about to be dropped, we can’t just
           clone it because there may be other things, such as tcpdump,
           interested in it). We also need to expand headroom in case
       hh_len of incoming interface < hh_len of outgoing interface */
    nskb = skb_copy_expand(oldskb, hh_len, skb_tailroom(oldskb),
                   GFP_ATOMIC);  // 用copy expand复制一份skb，和clone的区别在于，同时复制数据区，而clone只是增加data区引用计数。得到新的待发送的skb
    if (!nskb) {
        dst_release(&rt->u.dst);
        return;
    }

    dst_release(nskb->dst); // 释放老skb中的路由表引用
    nskb->dst = &rt->u.dst; // 使用新的翻转后的路由

    /* This packet will not be the same as the other: clear nf fields */
    nf_conntrack_put(nskb->nfct); // 很重要！必须释放conntrack结构，由于RST是由LOCAL发出，所以发出时不应该带conntrack，否则LOCAL_OUT 的conntrack hook会忽略这个包。
    nskb->nfct = NULL; // 清楚一些netfilter的成员
    nskb->nfcache = 0;
#ifdef CONFIG_NETFILTER_DEBUG
    nskb->nf_debug = 0;
#endif
    nskb->nfmark = 0;
    // 新的skb中tcp header
    tcph = (struct tcphdr *)((u_int32_t*)nskb->nh.iph + nskb->nh.iph->ihl);

    /* Swap source and dest */  // 交换tcp 源、目的端口号
    tmp_addr = nskb->nh.iph->saddr;
    nskb->nh.iph->saddr = nskb->nh.iph->daddr;
    nskb->nh.iph->daddr = tmp_addr;
    tmp_port = tcph->source;
    tcph->source = tcph->dest;
    tcph->dest = tmp_port;

    /* Truncate to length (no data) */  // 将tcp 数据区清空（RST包不需要带数据）
    tcph->doff = sizeof(struct tcphdr)/4;
    skb_trim(nskb, nskb->nh.iph->ihl*4 + sizeof(struct tcphdr));
    nskb->nh.iph->tot_len = htons(nskb->len);

   // 重新设置 seq ack_seq，分两种情况（TCP/IP详卷有描述）
    if (tcph->ack) {  // 异常关闭链接
        needs_ack = 0;
        tcph->seq = otcph->ack_seq;
        tcph->ack_seq = 0;
    } else { // 老skb为到端口不存在的链接请求
        needs_ack = 1;
        tcph->ack_seq = htonl(ntohl(otcph->seq) + otcph->syn + otcph->fin
                      + otcplen – (otcph->doff<<2));  // ack_seq为老skb数据包的seq+len
        tcph->seq = 0; // seq 默认为0
    }

    /* Reset flags */ // 设置tcp头RST标志
    ((u_int8_t *)tcph)[13] = 0;
    tcph->rst = 1;
    tcph->ack = needs_ack;

    tcph->window = 0;
    tcph->urg_ptr = 0;

    /* Adjust TCP checksum */
    tcph->check = 0;
    tcph->check = tcp_v4_check(tcph, sizeof(struct tcphdr),  // 重新计算tcp checksum
                   nskb->nh.iph->saddr,
                   nskb->nh.iph->daddr,
                   csum_partial((char *)tcph,
                        sizeof(struct tcphdr), 0));

    /* Adjust IP TTL, DF */
    nskb->nh.iph->ttl = MAXTTL;
    /* Set DF, id = 0 */
    nskb->nh.iph->frag_off = htons(IP_DF); // 不要分片
    nskb->nh.iph->id = 0;

    /* Adjust IP checksum */
    nskb->nh.iph->check = 0; // 重新计算ip checksum
    nskb->nh.iph->check = ip_fast_csum((unsigned char *)nskb->nh.iph, 
                       nskb->nh.iph->ihl);

    /* "Never happens" */
    if (nskb->len > nskb->dst->pmtu)
        goto free_nskb;

    connection_attach(nskb, oldskb->nfct);
    // 调用nf hook，把包发出去。这里会从LOCAL_OUT 到 POST_ROUTING。
    NF_HOOK(PF_INET, NF_IP_LOCAL_OUT, nskb, NULL, nskb->dst->dev,
        ip_finish_output);
    return;

 free_nskb:
    kfree_skb(nskb);
}

```
