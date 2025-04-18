---
title: "소프트웨어 응집도와 LCOM: 이해와 활용"
description: "소프트웨어 설계의 핵심 개념인 **응집도(Cohesion)** 를 다양한 유형과 예시를 통해 쉽게 이해할 수 있도록 설명합니다. 또한, 클래스의 응집도를 정량적으로 평가하는 LCOM(Lack of Cohesion in Methods) 메트릭을 활용하여 설계 품질을 분석하고 개선하는 방법을 제시합니다."
date: 2023-03-12T11:41:11+09:00
lastmod: 2024-12-29T18:15:00+09:00
url: "/cohesion-in-software-engineering/"
tags: [software engineering]
---

## 응집 (Cohesion)

응집은 소프트웨어 모듈 내 구성 요소들이 얼마나 밀접하게 연관되어 있는지를 나타내는 개념으로, **높은 응집도**는 모듈의 독립성과 유지보수성을 높이는 데 기여합니다. 응집은 여러 유형으로 나뉘며, 각 유형은 모듈 내 구성 요소들의 관계와 협력 방식을 기준으로 정의됩니다. 아래에서는 각 응집 유형을 **예시와 함께** 설명합니다.

### **1. 기능적 응집 (Functional Cohesion)**

모듈 내 모든 구성 요소가 **단일한 목적**을 위해 협력하며, 특정 작업을 완벽히 수행하기 위해 필요한 모든 기능이 포함된 경우입니다. 이는 가장 이상적인 응집 형태로 간주됩니다.

- **예시**:  
  계산기 프로그램에서 사각형의 넓이와 둘레를 계산하는 모듈은 입력값(가로, 세로)을 받아 넓이와 둘레를 계산한 후 결과를 반환합니다. 이 모듈은 단일 작업(사각형 계산)에 집중되어 있어 기능적 응집을 가집니다.

### **2. 순차적 응집 (Sequential Cohesion)**

모듈 내 구성 요소들이 **순차적으로 실행**되며, 하나의 구성 요소 출력이 다음 구성 요소의 입력으로 사용되는 경우입니다.

- **예시**:  
  데이터 처리 파이프라인을 생각해볼 수 있습니다. 예를 들어, 파일에서 데이터를 읽고 → 데이터를 정제하고 → 정제된 데이터를 데이터베이스에 저장하는 작업이 순서대로 이루어지는 모듈은 순차적 응집을 가집니다.

### **3. 소통적 응집 (Communicational Cohesion)**

모듈 내 구성 요소들이 **공통 데이터**를 사용하거나 동일한 데이터를 기반으로 작업하는 경우입니다.

- **예시**:  
  고객의 장바구니 데이터를 처리하는 모듈에서는 장바구니 데이터를 기반으로 할인 계산, 배송비 계산, 세금 계산 등의 작업이 이루어질 수 있습니다. 이처럼 구성 요소들은 서로 다른 작업을 수행하지만, 동일한 데이터를 공유하며 협력합니다.

### **4. 절차적 응집 (Procedural Cohesion)**

구성 요소들이 특정 절차나 **순서를 따라 실행**되도록 그룹화된 경우입니다. 이는 순차적 응집과 유사하지만, 반드시 동일한 데이터를 다루지 않을 수도 있습니다.

- **예시**:  
  사용자 인증 모듈에서 사용자의 자격 증명을 확인하고 → 액세스 토큰을 생성하며 → 사용자 활동 로그를 업데이트하는 과정은 절차적 응집의 예입니다.

### **5. 일시적 응집 (Temporal Cohesion)**

모듈 내 구성 요소들이 특정 시간이나 이벤트에 따라 함께 실행되는 경우입니다.

- **예시**:  
  시스템 초기화 시, 로그 파일 생성, 설정 파일 로드, 캐시 초기화 등 서로 연관성이 없어 보이는 작업들이 동시에 실행된다면 이는 일시적 응집에 해당합니다.

### **6. 논리적 응집 (Logical Cohesion)**

구성 요소들이 기능적으로 연관되기보다는 **논리적인 범주**에 따라 그룹화된 경우입니다.

- **예시**:  
  자바의 `StringUtils` 클래스처럼 문자열 관련 다양한 정적 메서드(문자열 대소문자 변환, 문자열 자르기 등)가 포함된 경우입니다. 이들은 논리적으로는 관련이 있지만, 기능적으로는 독립적입니다.

### **7. 동시적 응집 (Coincidental Cohesion)**

모듈 내 구성 요소들이 단순히 같은 소스 파일에 포함되어 있을 뿐, 서로 아무런 연관성이 없는 경우입니다. 이는 가장 낮은 수준의 응집도로 간주됩니다.

- **예시**:  
  한 모듈에 문자열 출력 함수와 리스트 정렬 함수가 함께 포함되어 있다면 이는 동시적 응집의 예로 볼 수 있습니다. 이러한 설계는 유지보수성과 재사용성을 저하시킵니다.

---

## LCOM (Lack of Cohesion in Methods)

코드의 응집도를 정량적으로 평가하기 위해 LCOM(Lack of Cohesion in Methods) 메트릭을 사용할 수 있습니다. 이는 클래스 내 메서드들이 공유 필드를 얼마나 활용하는지 분석하여 응집도를 측정합니다.

### LCOM 활용 예: 클래스 X, Y, Z 비교

![LCOM 메트릭](/images/LCOM.jpg)

이미지에 나타난 클래스 X, Y, Z는 LCOM(Lack of Cohesion in Methods, 메서드 간 응집 결여도)을 계산하고 이해하는 데 유용한 사례를 제공합니다. 각 클래스의 구조를 분석하여 응집도를 평가해보겠습니다.

#### **1. 클래스 X**
- **구성**:
    - 필드 A, B, C (육각형)
    - 메서드 m1(), m2(), m3() (사각형)
    - 각 메서드는 여러 필드를 공유하며 서로 연결되어 있습니다.

- **분석**:  
  클래스 X는 모든 필드(A, B, C)가 여러 메서드(m1(), m2(), m3())에 의해 공유되고 사용됩니다. 이는 메서드와 필드가 서로 밀접하게 연관되어 있음을 나타냅니다.

- **LCOM 평가**:  
  LCOM 점수가 낮습니다(즉, 응집도가 높음). 이는 클래스가 단일한 목적을 가지고 있으며, 메서드들이 협력하여 작업을 수행한다는 것을 보여줍니다.

#### **2. 클래스 Y**
- **구성**:
    - 필드 A, B, C
    - 메서드 m1(), m2(), m3()
    - 각 필드는 오직 하나의 메서드에서만 사용됩니다.

- **분석**:  
  클래스 Y에서는 각 메서드가 특정 필드만 사용하며 다른 필드나 메서드와 상호작용하지 않습니다. 이는 메서드들이 독립적으로 작동한다는 것을 의미합니다.

- **LCOM 평가**:  
  LCOM 점수가 매우 높습니다(즉, 응집도가 낮음). 이 경우 클래스 Y는 단일 클래스로 유지할 필요가 없으며, 각 필드와 관련된 메서드를 별도의 클래스로 분리하는 것이 더 바람직합니다.

#### **3. 클래스 Z**
- **구성**:
    - 필드 A, B, C
    - 메서드 m1(), m2(), m3()
    - 일부 메서드는 여러 필드를 공유하며 상호작용하지만, 특정 필드는 독립적으로 사용됩니다.

- **분석**:  
  클래스 Z는 일부 메서드와 필드가 서로 연관되어 있지만, 다른 구성 요소들은 독립적으로 작동합니다. 이는 클래스 내에서 응집도가 부분적으로 유지되고 있음을 나타냅니다.

- **LCOM 평가**:  
  LCOM 점수는 중간 수준입니다. 독립적인 구성 요소(예: C와 관련된 m3())를 별도의 클래스로 분리하면 응집도를 향상시킬 수 있습니다.

#### **결론**
- **클래스 X**는 높은 응집도를 가지며 잘 설계된 구조입니다.
- **클래스 Y**는 낮은 응집도를 가지며, 리팩토링을 통해 각 필드와 관련된 메서드를 별도의 클래스로 분리하는 것이 적합합니다.
- **클래스 Z**는 부분적으로 응집되어 있으며, 독립적인 구성 요소를 분리하여 개선할 여지가 있습니다.

이처럼 LCOM 분석은 클래스의 구조적 결함을 식별하고 설계를 개선하는 데 유용한 도구로 활용될 수 있습니다.

---

## 결론

응집도는 소프트웨어 설계 품질을 평가하는 중요한 척도이며, 높은 응집도를 유지하는 것이 바람직합니다. 각 유형의 응집을 이해하고 이를 설계에 적용함으로써 더 나은 모듈화와 유지보수성을 달성할 수 있습니다.
