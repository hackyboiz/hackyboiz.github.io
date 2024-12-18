---
title: "[하루한줄] CVE-2024-22107: GTB Central Console Command Injection 취약점"
author: romi0x
tags: [GTB Central Console,Command Injection, PHP, romi0x]
categories: [1day1line]
date: 2024-12-18 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

- https://nvd.nist.gov/vuln/detail/CVE-2024-22107

## Target

- **Target Product**: GTB Central Console
- **Affected Versions**: 15.17.1-30814.NG (포함)

## Explain
GTB Central Console은 기업의 데이터 보안을 담당하는 데이터 손실 방지(DLP) 소프트웨어로, 파일, 이메일, 네트워크 트래픽 등에서 민감한 데이터를 보호합니다.

![GTB 사진](1day1line1218/image.png)


취약점은 `/opt/webapp/src/AppBundle/Controller/React/SystemSettingsController.php` 파일 내의 `systemSettingsDnsDataAction` 메서드에서 발생합니다.  해당 메서드는외부로부터 입력된 데이터에 대해 제대로 검증하지 않아  공격자가 임의의 명령을 실행할 수 있습니다.

다음은 코드 일부분입니다.

```php
//...
public function systemSettingsDnsDataAction(Request $request)
    {
        /** @var ConfigurationNetworkHandler $cnh */
        $cnh     = $this->container->get('gtb.handler.configuration_network');
        $xaction = $request->request->get('xaction', null);
        if (!$xaction) {
            $content = json_decode($request->getContent(), true);
            $xaction = $content['xaction'];
        }

        /** @var Translator $translator */
        $translator = $this->container->get('translator');

        $ssRepo = $this->getDm()->getRepository('AppBundle:SystemSettings');

        switch ($xaction) {
            case 'read':
                $dns  = $cnh->getDnsServer();
                $data = [
                    'dnsServerIps' => $dns,
                    'cc_name'      => $ssRepo->getParameterByName(SystemSettings::PARAM_CC_NAME)->getValue(),
                    'host_name'    => trim(file_get_contents('/etc/hostname'))
                ];

                return new JsonResponse([
                    'results' => $data
                ]);
            case 'update':
                $data   = json_decode($request->request->get('data', '{}'), true);
                $status = false;

                if (isset($data['dnsServerIps'])) {
                    if (!$this->isMultiTenant()) {
                        $cnh->setDnsServer($data['dnsServerIps']);
                    }
                    $ssRepo->setParameterByName(SystemSettings::PARAM_CC_NAME, $data['cc_name']);
                    if (!$this->isMultiTenant()) {
                        exec('sudo /opt/webapp/shell/set_hostname.sh ' . $data['host_name']);
                    }
//...
```
이 코드를 보면 `host_name` 파라미터가 필터링 없이 `exec()` 함수로 전달된것이 문제가 됩니다.

`exec('sudo /opt/webapp/shell/set_hostname.sh ' . $data['host_name']);`

서버의 네트워크 정보를 수집하여 파일로 저장하는 Command Injection 페이로드의 예시로는 다음 코드와 같습니다.

```php
{
    "dnsServerIps": "8.8.8.8",
    "cc_name": "",
    "host_name": "target; ifconfig > /tmp/network_info.txt"
}
```

취약점을 악용한  공격자는 서버를 완전히 장악하거나 악성 행위를 할 수 있습니다.
## Reference

- https://adepts.of0x.cc/gtbcc-pwned/