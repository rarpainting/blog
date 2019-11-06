# Java

## 坑

### `ClassNotFoundException: javax.xml.bind.JAXBContext`

[stackoverflow](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception-in-j)

运行 springboot 项目出现: `Type javax.xml.bind.JAXBContext not present`

原因: java9+ 版本以后, JAXB 默认没有加载(但是在测试中, Java8 也没有启用 jaxb, 为什么 ?)

解决方案:
```xml
<!-- spring-boot 1.5.* -->
<!-- Java 6 = JAX-B Version 2.0   -->
<!-- Java 7 = JAX-B Version 2.2.3 -->
<!-- Java 8 = JAX-B Version 2.2.8 -->
<dependency>
    <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-core</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>2.2.11</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
```

```xml
<!-- spring boot 2.0.* -->
<dependency>
   <groupId>javax.xml.bind</groupId>
    <artifactId>jaxb-api</artifactId>
</dependency>
<dependency>
    <groupId>com.sun.xml.bind</groupId>
    <artifactId>jaxb-impl</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
    <version>2.3.0</version>
</dependency>
<dependency>
    <groupId>javax.activation</groupId>
    <artifactId>activation</artifactId>
    <version>1.1.1</version>
</dependency>
```
