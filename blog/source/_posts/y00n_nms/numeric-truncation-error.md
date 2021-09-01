---
title: "[해키피디아] Numeric Truncation Error"
author: y00n_nms
tags: [y00n_nms, numeric truncation error]
categories: [Hackypedia]
date: 2021-09-01 14:00:00
cc: true
index_img: /img/hackypedia.png

---

Numeric Truncation Error는 기본 요소가 더 작은 크기의 기본 요소로 변환되는 과정에서 데이터가 손실되어 원래 값과 같지 않은 예상하지 못한 값으로 저장될 때 발생합니다.

프로그래머는 당연히 정수 변환 및 계산이 정확하고 수학 법칙을 준수할 것이라고 가정하지만, 컴퓨터는 일반적으로 정수를 고정 길이 비트에 포함된 이진값으로 표현하기 때문에 오류가 발생할 기회가 있습니다.

예를 들어 16bit로 숫자 256을 표현한다면 `000000100000000` 으로 1 다음에 오른쪽에 8개의 0으로 표현됩니다. 이를 8bit로 변환하게 되면 상위 8bit가 손실되어 `00000000`만 남습니다.

```c
int intPrimitive;
short shortPrimitive;
intPrimitive = (int)(~((int)0) ^ (1 << (sizeof(int)*8-1)));
shortPrimitive = intPrimitive;
printf("Int MAXINT: %d\\nShort MAXINT: %d\\n", intPrimitive, shortPrimitive);
```

```c
Int MAXINT: 2147483647
Short MAXINT: -1
```

signed int의 범위는 32bit로 2,147,483,648 ~ 2,147,483,647까지 표현 가능하고 signed short는 16bit로 32,768 ~ 32,767까지 표현 가능합니다. `shortPrimitive = intPrimitive;` 줄로 int가 short로 변환되어 저장됩니다.

2147483647인 `01111111111111111111111111111111`가 16bit로 변환되면 상위 16bit가 손실되어 `1111111111111111`이 남습니다. 따라서 shortPrimitive에 예상한 값이 아닌 -1이 저장됩니다.

![image](numeric-truncation-error/image.png)

이 예상하지 못한 값이 배열 인덱스, 루프 카운터 또는 상태 데이터로 사용될 때 예상하지 못한 취약점이 추가로 발생할 수 있기 때문에 주의해야 합니다.

다음 Java 예제에서 `updateSalesForProduct` 메소드는 특정 제품에 대한 판매 정보를 업데이트하는 비즈니스 애플리케이션 클래스의 일부입니다. 이 메서드는 제품 ID와 판매된 금액을 인수로 받습니다. 제품 ID는 개수를 정수로 반환하는 인벤토리 개체에서 총 제품 개수를 검색하는 데 사용됩니다. 판매 수를 업데이트하기 위해 `sales` 개체의 메서드를 호출하기 전에 메서드 인수에 대해 short 형식이 필요하므로 int 값이 short로 변환됩니다.

```java
...
// update sales database for number of product sold with product ID 
public void updateSalesForProduct(String productID, int amountSold) {

// get the total number of products in inventory database 
int productCount = inventory.getProductCount(productID);
// convert integer values to short, the method for the 

// sales object requires the parameters to be of type short 
short count = (short) productCount;
short sold = (short) amountSold;
// update sales database for product 
sales.updateSalesCount(productID, count, sold);
}
...
```

이때 정수 값이 short에 허용되는 최댓값보다 크면  Numeric Truncation Error가 발생할 수 있습니다. 이로 인해 예기치 않은 결과가 발생하거나 데이터가 손실되거나 손상되어 이 예시의 경우 판매 데이터베이스가 잘못된 데이터로 손상될 수 있습니다.

Numeric Truncation Error를 방지하기 위해서는 더 큰 크기의 기본 요소에서 더 작은 크기의 기본 요소으로의 명시적 형변환이 발생하지 않도록 구현하거나 다음 예제처럼 명시적 형변환 및 sales 메서드 호출 이전에 기본 형식의 최댓값보다 작은 정수값이 있는지 확인하는 if 문을 추가로 작성해야 합니다.

```java
// maximum value for type short before converting 
if ((productCount < Short.MAX_VALUE) && (amountSold < Short.MAX_VALUE)) { ... }
```