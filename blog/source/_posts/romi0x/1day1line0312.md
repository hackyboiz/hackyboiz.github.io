---
title: "[하루한줄] CVE-2024-1086: Linux Kernel 내 Use-After-Free"
author: romi0x
tags: [Linux, UAF, netfilter, kernel, romi0x]
categories: [1day1line]
date: 2025-03-12 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

https://www.igloo.co.kr/security-information/linux-kernel-%eb%82%b4-use-after-free-%ec%b7%a8%ec%95%bd%ec%a0%90cve-2024-1086-%eb%b6%84%ec%84%9d-%eb%b0%8f-%eb%8c%80%ec%9d%91%eb%b0%a9%ec%95%88/

## Target

- Linux Kernel 6.1 ~ 6.5 버전

## Explain

이 취약점은 Linux Kernal 내 리눅스 패킷 필터링 및 네트워크 주소 변환(NAT) 프레임워크인 netfilter의 nf_tables 구성 요소에서 UAF가 발생하는 취약점입니다.

이 취약점은 nftables의 패킷 처리 과정에서 발생하며,  `nf_hook_slow()`와 `nft_verdict_init()` 함수에서 발생합니다.

아래는 `nf_hook_slow()` 함수의 일부입니다.

```
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state, 
                 const struct nf_hook_entries *e, unsigned int s)
{
    unsigned int verdict;
    int ret;

    for (; s < e->num_hook_entries; s++) {
        verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
        
        switch (verdict & NF_VERDICT_MASK) {
            case NF_ACCEPT:
                break;

            case NF_DROP:
                kfree_skb(skb);
                ret = NF_DROP_GETERR(verdict);
                if (ret != 0)  // 오타 수정
                    ret = -EPERM;
                return ret;

            case NF_QUEUE:
                ret = nf_queue(skb, state, s, verdict);
                if (ret == 1)
                    continue;
                return ret;

            default:
                break;
        }
        return 0;
    }
    return 1;
}
```

위 코드는 netfilter 모듈 내의 `nf_hook_slow()` 함수의 일부로, 패킷 처리 규칙을 반복문 안에서 평가합니다. 패킷에 대한 verdict 값을 얻고, 그 값과 NF_VERDICT_MASK 매크로 값을 통해 패킷 처리를 결정합니다. verdict 값은 패킷 처리 결과값을 나타내며, 이는 사용자가 설정할 수 있는 값입니다.

`nf_hook_slow()` 함수에서는 NF_DROP일 경우 `kfree_skb()`를 호출하여 패킷을 해제합니다. 이후 NF_DROP_GETERR(verdict)에서 공격자가 설정한 verdict 값이 "0xFFFF0000"이라면, 함수 내부 연산을 통해 ret 값은 -65535로 설정됩니다. 이 값은 이후 NF_ACCEPT로 처리되어 패킷이 계속해서 처리됩니다.

아래는 `nft_verdict_init()` 함수입니다.

```
static int nft_verdict_init(const struct nft_ctx *ctx, struct nft_data *data, struct nft_data_desc *desc, const struct_nlattr *nla)
{
    switch (data->verdict.code) {
    default:
        switch (data->verdict.code & NF_VERDICT_MASK) {
        case NF_ACCEPT:
        case NF_DROP:
        case NF_QUEUE:
            break;
        default:
            return -EINVAL;
        }
    }

    return 0;
}
```

`nft_verdict_init()` 함수에서 `data->verdict.code`는 규칙 설정에 대한 리턴 값으로 사용되며, 공격자가 "0xFFFF0000" 값을 설정할 경우 취약점이 발생합니다.

NF_VERDICT_MASK 매크로를 통해 verdict.code의 하위 16비트를 추출하면, "0xFFFF0000 & 0x0000FFFF"가 되어 "0x00000000"이 됩니다. 이 값은 NF_DROP을 의미하기 때문에 해당 패킷을 드롭하는 것으로 처리됩니다. 그러나 `nf_hook_slow()` 함수에서 패킷을 드롭하려는 처리 후, NF_DROP_GETERR(verdict) 연산에 의해 ret 값은 -65535로 설정되며, NF_ACCEPT로 처리됩니다.

이로 인해 이미 `kfree_skb()`로 해제된 패킷이 다시 처리되어 이전에 해제된 소켓 메모리(skb)를 참조하게 되어 Use-After-Free(UAF) 상황이 발생하고, 그 후 다시 소켓 메모리를 해제하면서 이중 해제(double-free)가 발생하게 됩니다.

패치된 `nft_verdict_init()` 함수는 다음과 같습니다:

```
switch (data->verdict.code) {
  case NF_ACCEPT:
  case NF_DROP:
  case NF_QUEUE:
     break;
  case NFT_CONTINUE: 
  case NFT BREAK:
  case NFT_RETURN:
     data->verdict.chain-chain;
     break;
  default:
     return -EINVAL;
  }
```

`data->verdict.code` 값에 대해 검증이 추가되었습니다. 이제 `verdict.code` 값은 허용된 값들만 처리하고, 잘못된 값이 들어오면 무조건 `-EINVAL`을 반환하여 악의적인 입력을 차단합니다. 이로 인해 패킷 처리가 정상적으로 이루어지며, 취약점을 방지할 수 있습니다.

결론적으로, 이 취약점은 `nf_hook_slow()`에서 패킷을 처리하고, 잘못된 verdict 값으로 인해 이미 해제된 메모리를 참조하게 되어 Use-After-Free(UAF)와 이중 해제(double-free) 문제가 발생하는 것입니다. 패치된 버전에서는 verdict.code 값을 직접 검증하여 유효하지 않은 값에 대해 오류를 반환함으로써 이 문제를 해결합니다.

## Reference

https://nvd.nist.gov/vuln/detail/cve-2024-1086