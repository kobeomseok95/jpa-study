Querydsl 시작하기
=========================

우선 bulid.gradle로 들어가자

내 실습환경의 버전 정보

- spring boot 2.4.1
- h2 database 1.4.200
- Gradle 6.7.1
- java 11

1. Plugin

   ```groovy
   plugins {
   	...
   	id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
   	...
   }
   ```

2. dependencies

   ```groovy
   dependencies {
   	...
   	implementation 'com.querydsl:querydsl-jpa'
   	...
   }
   ```

3. Querydsl이 생성하는 QClass들의 경로 설정

   ```groovy
   def querydslDir = "$buildDir/generated/querydsl"
   
   querydsl {
   	jpa = true
   	querydslSourcesDir = querydslDir
   }
   sourceSets{
   	main.java.srcDir querydslDir
   }
   configurations {
   	querydsl.extendsFrom compileClasspath
   }
   compileQuerydsl {
   	options.annotationProcessorPath = configurations.querydsl
   }
   ```

4. 빌드 후 엔티티를 작성한 다음 오른쪽에서 gradle > tasks > other > compileQuerydsl을 클릭

![image](https://user-images.githubusercontent.com/37062337/103392204-081d1780-4b60-11eb-8fe1-e0582dab0623.png)

5. querydslDir 경로에 QType의 클래스가 생성된 것을 확인할 수 있다. 이 build 폴더는 ignore해주자

![image](https://user-images.githubusercontent.com/37062337/103392275-62b67380-4b60-11eb-87d3-fec64a27f435.png)

