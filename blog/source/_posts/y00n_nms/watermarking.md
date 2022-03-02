---
title: "[해키피디아] WaterMarking VS. FingerPrinting"
author: y00n_nms
tags: [y00n_nms, watermarking, fingerprinting, drm]
categories: [Hackypedia]
date: 2022-03-02 14:00:00
cc: true
index_img: /img/hackypedia.png

---

워터마킹(WaterMarking)과 핑거프린팅(FingerPrinting)은 디지털 콘텐츠 저작권 소유자의 권리를 보호하고 콘텐츠가 무단 배포 및 수정되는 것을 방지하기 위한 보안 기술인 **[DRM(Digital Right Management)](https://hackyboiz.github.io/2022/02/23/y00n_nms/drm/) ** 중 일부입니다.

**워터마킹**은 볼법 복제를 방지하기 위하여 디지털 콘텐츠가 최초 만들어질 시점에 저작권(저작권자) 정보를 삽입합니다.

삽입된 워터마크의 강인성에 따라서 활용 용도가 조금 달라지게 됩니다. 원 저작물의 데이터를 파괴하지 않고서는 워터마크를 지울 수 없는 강성 워터마킹은 소유권 증명에 사용되며, 약간의 변형에도 워터마크가 수정되거나 사라지게 되는 연성 워터마킹은 위조, 변조가 발생했는지 파악할 수 있으며 및 무결성 확보에 사용됩니다.

또한 인지 여부, 삽입 방법 등 여러 기준으로 구분할 수 있습니다. 워터마킹을 인지할 수 있도록(눈에 보이게) 삽입하는 가시적 방법과 인지할 수 없도록 삽입하여 특별한 과정을 거쳐야 저작권을 확인할 수 있는 비가시적 방법이 있고, 삽입 방법에 따라 LSB(Least Significant Transform)값을 수정하는 공간 영역 방법과 FFT(Fast Fourier Transform)이나 DCT(Discrete Cosine Transform)을 수정하는 주파수 변환 방법이 있습니다.

삽입되는 정보가 저작권(저작권자)인 워터마킹에 비해 **핑거프린팅**은 불법 유통을 방지하기 위하여 구매할 때마다 저작권 정보와 구매자 정보까지 삽입됩니다. 즉, 콘텐츠를 불법적으로 사용하거나 불법으로 유통하게 된다면 유포자의 정보를 파악할 수 있습니다. 실제로 [웹툰 플랫폼 레진코믹스는 콘텐츠를 불법 유통했던 ‘밤토끼' 운영자를 핑거프린팅으로 검거할 수 있었습니다.](https://www.sedaily.com/NewsVIew/1RZN7ZYAQK) 하지만 최초 유포자만 확인이 가능하고 2차, 3차 유포에 대해서는 확인할 수 없습니다.