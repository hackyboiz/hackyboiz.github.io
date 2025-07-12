---
title: "[하루한줄] CVE-2025-27363: FreeType의 Out-of-Bounds Write 취약점"
author: romi0x
tags: [FreeType, Out-of-Bounds Write, Android, romi0x]
categories: [1day1line]
date: 2025-06-04 17:00:00
cc: true
index_img: /img/1day1line.png
---

## URL

https://nvd.nist.gov/vuln/detail/CVE-2025-27363

## Target

- FreeType 버전 2.13.0 이하

## Explain

FreeType은 다양한 운영체제와 애플리케이션에서 사용되는 오픈소스 폰트 렌더링 라이브러리입니다. TrueType, OpenType, CFF 등 여러 포맷을 지원하며, Android, Linux 데스크탑, 게임 엔진 등에서도 널리 사용됩니다.

CVE-2025-27363은 TrueType GX 및 variable font 처리 중 발생하는 **Out-of-Bounds Write** 취약점입니다. 이는 FreeType이 폰트 내부의 서브 글리프(subglyph, 글리프를 이루는 작은 조각)를 처리하는 과정에서 발생합니다. 폰트에는 `%`, `&`, `@` 같은 글자가 단일 도형이 아니라 여러 작은 도형들로 조립되어 있습니다. 이 각각의 조각들을 subglyph라고 합니다.

취약점은 FreeType의 `load_truetype_glyph()` 함수에서 글리프에 포함된 구성 요소 수를 계산할 때 정수 오버플로우가 발생하여, 실제 필요한 메모리보다 작게 힙 메모리가 할당되면서 발생합니다. 이후 많은 구성 요소를 메모리에 쓰려고 할 때 할당된 버퍼 범위를 넘어서는 힙 버퍼 오버플로우가 발생하게 됩니다.

참고한 [PoC](https://github.com/zhuowei/CVE-2025-27363-proof-of-concept)에서는 Google의 Roboto Flex 폰트를 기반으로 `%` 글리프에 65,536개의 구성 요소 글리프를 삽입해 취약점을 트리거합니다.

```
from fontTools.ttLib import TTFont
from fontTools.ttLib.tables._g_l_y_f import GlyphComponent, Glyph

font = TTFont("RobotoFlex.ttf")
glyph = Glyph()
glyph.components = []

for _ in range(0x10000):  # 65,536개의 서브글리프 추가
    comp = GlyphComponent()
    comp.glyphName = "space"
    comp.x = 0
    comp.y = 0
    glyph.components.append(comp)

font["glyf"].glyphs["percent"] = glyph
font.save("rf2.ttf")

```

패치는 FreeType 소스코드의 `ttgload.c` 파일 내 `load_truetype_glyph()` 함수에서 이루어졌으며, 정수 오버플로우를 방지하기 위한 범위 체크와 안전한 메모리 할당 매크로(`FT_QALLOC_MULT`)를 사용하도록 수정되었습니다.

- 수정 전

    ```
    n_subglyphs = num_subglyphs + 4;
    loader->extra_points = (FT_Vector*)ft_mem_alloc(n_subglyphs * sizeof(FT_Vector));
    ```

  `num_subglyphs + 4` 계산 과정에서 정수 오버플로우가 발생하면 값이 작은 수로 순환(wrap-around)되어, 힙 메모리가 실제 필요한 크기보다 적게 할당됩니다. 이로 인해 이후에 많은 구성 요소를 기록하려 할 때 버퍼를 넘는 오버플로우가 발생합니다.

- 수정 후

    ```
    if ( num_subglyphs > SOME_MAX_LIMIT )
      return FT_THROW( Invalid_Outline );
    
    n_subglyphs = num_subglyphs + 4;
    
    if ( FT_QALLOC_MULT( loader->extra_points, n_subglyphs, sizeof(FT_Vector) ) )
      return FT_THROW( Out_Of_Memory );
    ```

  덧셈 전에 범위 검사를 추가하여 정수 오버플로우를 예방하고, `FT_QALLOC_MULT()` 매크로를 사용해 안전한 메모리 할당을 수행합니다. 이를 통해 정수 오버플로우, 메모리 과소할당, Out-of-Bounds Write를 방지했습니다.

## Reference

https://android.googlesource.com/platform/external/freetype/+/f9c146bda91f441b7bebac4387e46ae40caaf3b1

https://github.com/zhuowei/CVE-2025-27363-proof-of-concept