---
layout: post
title: "Python에서 Java Class 호출(Jython 사용안함)"
date: 2022-05-16
author: 손정모

---

이번 포스팅에서는 Java Class를 Python에서 호출하여 사용하는 방법에 대해 알아 보겠습니다.   
pyjnius 라이브러리를 사용하여 Python에서 호출하는 방법입니다.    
아래의 예제는 PyTest.class를 Python에서 직접 호출하는 것입니다.   
    
## 순서   
1. pyjnius 설치    
2. PyTest.java 작성 및 컴파일    
3. python에서 pyjnius를 호출하기 위한 환경변수 설정     
4. python에서 PyTest를 호출     
     
### 1. pyjnius 설치
pyjnius는 아래의 명령어로 간편하게 설치 가능합니다.   
    
```
pip install pyjnius
```    
    
### 2. PyTest.java 작성 및 컴파일
python에서 호출할 java 프로그램 입니다. 테스트용 메소드는 create 메소드로 HashMap을 입력받아 값을 추가하고 다시 반환하는 형태입니다.     
    
```java
package com.test;

import java.util.HashMap;

public class PyTest {
	
	public HashMap<String, String> create(HashMap<String, String> map) {
		
		// 새로운 값 추가
		map.put("ADD", "Hello");
		
		// map의 내용 출력
		for(String key: map.keySet()) {
			System.out.println(key + ":" + map.get(key));
		}
		
		return map;
	}
	
}
```    
     
PyTest.java compile    
```
javac -d . PyTest.java
```    
![image](1)    
    
### 3. python에서 pyjnius를 호출하기 위한 환경변수 설정     
JAVA가 설치되어 있는 디렉토리를 JAVA_HOME 환경 변수로 설정해야 함     
```
C:\app\test>set JAVA_HOME=C:\Program Files\ojdkbuild\java-1.8.0-openjdk-1.8.0.282-1
```    
    
### 4. python에서 PyTest를 호출     
pyjnius 라이브러리를 이용하여, python에서 PyTest 호출하고 결과를 화면에 출력함
   
test.py   
```python
# jnius의 JVM 설정
import jnius_config
jnius_config.set_classpath(r'C:\app\test')

# jnius를 사용하여 java class 호출
# !주의) 아래의 jnius import 문장은 항상 jnius_config의 설정 뒤에 나와야 함
#       만일 먼저 나올 경우, 이미 JVM이 기동중이라 설정이 불가능하다는 오류가 발생함
from jnius import autoclass

# 해시맵 객체 생성 및 데이터 입력
hashmap_cls = autoclass('java.util.HashMap')
test_map = hashmap_cls()
test_map.put('MSG', r'Hello World!')

# 클래스 생성
pytest_cls = autoclass('com.test.PyTest')
pytest = pytest_cls()

# 메소드 호출 및 결과 출력
print('java에서 출력')
result = pytest.create(test_map)

print('')
print('python에서 출력')
print(result.toString())
```       
    
실행 결과     
![image](2)     
     
     
감사합니다.     

--------------------------
## References     
https://buildmedia.readthedocs.org/media/pdf/pyjnius/latest/pyjnius.pdf    
