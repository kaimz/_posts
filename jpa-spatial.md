---
title: Spring Data JPA 中使用空间数据
date: 2018-5-4 22:08:03
tags: [jpa,Spring]
categories: 学习笔记
---

### 前言
JPA 中使用空间数据字段的时候，出现了很多问题，中间走了好多弯路，这里记录下。

环境：
* Postgresql 9.5 + PostGIS
* JPA
* Spring 4.2

<!--more-->
### 使用
首先需要检查项目中的 hibernate 的版本依赖的问题，因为我们需要使用 `hibernate-spatial `[^1]的依赖包，它依赖 `hibernate-core 5.0` 以上的。还需要注意的一点是，我中间成了一个问题：

[^1]: Hibernate Spatial是对处理空间数据的一个Hibernate扩展 ，要想使用 Hibernate Spatial  就要引入JTS，完成Java 对几何图形对象的算法，


```java
NoSuchMethodError: org.hibernate.Session.getFlushMode()Lorg/hibernate/FlushMode;
```

给出的解释是 :

> `Hibernate 5.2` supports added in `Spring framework 4.3` , that its stable version will be available in next week. Spring 4.2 only supports Hibernate up to 5.1.

项目中的 使用的是 **Spring 4.2** ，只好把 Hibernate 降级到 5.1。

1. 加入 `hibernate-spatial` 依赖：

   ```xml
           <dependency>
               <groupId>org.hibernate</groupId>
               <artifactId>hibernate-spatial</artifactId>
               <version>5.1.0.Final</version>
           </dependency>
   ```

   hibernate已经可以支持空间数据类型的数据操作了，可以实现空间数据入库到空间数据对象影射到java对象中（插入和查询方法），例如 equals,contains,disjoint,within 等等。

2. 更改数据库方言：

   ```properties
   hibernate.dialect=org.hibernate.spatial.dialect.postgis.PostgisPG9Dialect
   ```

3. 空间数据库字段的映射：
   这里只做了一个点的字段；

   ```java
   	@Column(name = "geom", columnDefinition = "geometry(Point,4326)")
   	private Point geom;
   ```

4. 其余的 Repository 和 Service 跟普通无区别，

   保存数据：

   ```java
           Geometry geom = new WKTReader().read("POINT (-122.2985 47.6448)");
           Point point = geom.getInteriorPoint();
           point.setSRID(4326);
           reference.setGeom(point);
           repository.save(reference);
   ```

查询数据的时候，会出现一个问题，空间字段的json 中会出现：

```json
      "coordinates" : {
        "envelope" : {
          "envelope" : {
            "envelope" : {
                ……
```

这并不是我们需要的结果，原因是 *Jackson* 无法管理 `Point` 数据类型的序列化，需要我们自己定义它的序列化。

下面针对空间字段 `Point` 做一个序列化转换器。

1. 创建一个序列化转换器：

   ```java
   import com.fasterxml.jackson.core.JsonGenerator;
   import com.fasterxml.jackson.core.JsonProcessingException;
   import com.fasterxml.jackson.databind.JsonSerializer;
   import com.fasterxml.jackson.databind.SerializerProvider;
   import com.vividsolutions.jts.geom.Point;

   import java.io.IOException;

   public class PointToJsonSerializer extends JsonSerializer<Point> {
       @Override
       public void serialize(Point value, JsonGenerator gen, SerializerProvider serializers) throws IOException, JsonProcessingException {
           String jsonValue = "null";
           try
           {
               if(value != null) {
                   double lat = value.getY();
                   double lon = value.getX();
                   // wkt 格式，经纬度格式按照自己的需求
                   jsonValue = String.format("POINT (%s %s)", lon, lat);
               }
           }
           catch(Exception e) {
               // 不处理
           }
           gen.writeString(jsonValue);
       }
   }
   ```

2. 我们还需要从前端接收 geometry 字段的 wkt 形式的 json 字符串，这里需要反序列化。

   ```java
   import com.fasterxml.jackson.core.JsonParser;
   import com.fasterxml.jackson.core.JsonProcessingException;
   import com.fasterxml.jackson.databind.DeserializationContext;
   import com.fasterxml.jackson.databind.JsonDeserializer;
   import com.vividsolutions.jts.geom.Coordinate;
   import com.vividsolutions.jts.geom.GeometryFactory;
   import com.vividsolutions.jts.geom.Point;
   import com.vividsolutions.jts.geom.PrecisionModel;

   import java.io.IOException;

   public class JsonToPointDeserializer extends JsonDeserializer<Point> {

       private final static GeometryFactory GEOMETRY_FACTORY = new GeometryFactory(new PrecisionModel(), 4326);

       @Override
       public Point deserialize(JsonParser jp, DeserializationContext ctxt)
               throws IOException, JsonProcessingException {

           try {
               String text = jp.getText();
               if (text == null || text.length() <= 0) {
                   return null;
               }
               String[] coordinates = text.replaceFirst("POINT ?\\(", "").replaceFirst("\\)", "").split(" ");
               double lat = Double.parseDouble(coordinates[0]);
               double lon = Double.parseDouble(coordinates[1]);
               return GEOMETRY_FACTORY.createPoint(new Coordinate(lat, lon));
           }
           catch(Exception e){
               return null;
           }
       }
   }
   ```

3. 在空间字段上配置自定义的序列化，最终结果：

   ```java
   	@Column(name = "geom", columnDefinition = "geometry(Point,4326)")
   	@JsonSerialize(using = PointToJsonSerializer.class)
   	@JsonDeserialize(using = JsonToPointDeserializer.class)
   	private Point geom;
   ```

4. 测试向服务端传输数据和获取到的空间数据。

   * 返回的数据 `"geom":"POINT (-122.2985 47.6448)"`
   * 传输数据 POST 提交：
   
     ![image](https://zqnight.gitee.io/kaimz.github.io/image/hexo/junit-test/2.png)
     

这样做的一个问题就是其余的例如线和多边形，要重新新建序列化代码。

### 总结

这样就完成了自动的后端 `Geometry` 字段和  `wkt` 字符串数据的转换，同理其他形式的 ，例如`geojson` 格式一样可以进行自动转换。

补充一点，hibernate的postgis支持不是很好，国内资料比较少，很多需求或者问题都不是很好解决，而且它支持的 postgis 的函数库也比较少，所以可以使用原生 SQL 进行查询装载对象也是比较好的解决办法，例如使用 Mybatis。

### 参考文章
* [利用hibernate-spatial让Spring Data JPA支持空间数据](http://wiselyman.iteye.com/blog/2381465)
* [Map a PostGIS geometry point field with Hibernate on Spring Boot](https://stackoverflow.com/questions/27624940/map-a-postgis-geometry-point-field-with-hibernate-on-spring-boot)
