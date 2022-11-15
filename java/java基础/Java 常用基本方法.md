## 1、数组

### 1.1 数组快速初始化
```java
int[] arr = new int[4];
Arrays.fill(arr, -1);
```



## 2、集合

### 2.1 序列转集合
```java
List<Integer> aList = Arrays.asList(1,2,3);

int[] arr = new int[]{1,2,3};
aList = Arrays.asList(arr);
```

### 2.2 集合打印

```java
List<Integer> aList = Arrays.asList(1,2,3);
// 注意：需要现转换为String
String str = aList.stream()
			.map(String::valueOf)
			.collect(Collectors.joining(","));
System.out.println(str)
```


## 3、日期时间

### 3.1 LocalDateTime 转 LocalDate
  
```java
LocalDateTime localDateTime = LocalDateTime.now();
LocalDate localDate = localDateTime.toLocalDate();
```


### 3.2 LocalDate 转 LocalDateTime

```java
LocalDate localDate = LocalDate.now();
LocalDateTime localDateTime1 = localDate.atStartOfDay();
LocalDateTime localDateTime2 = localDate.atTime(8,20,33);
LocalDateTime localDateTime3 = localDate.atTime(LocalTime.now());
```


### 3.3 LocalDateTime, Date 相互转换

```java
// 先转成字符串，然后再进行转换
LocalDateTime localDateTime = LocalDateTime.now();
//获取系统默认时区
ZoneId zoneId = ZoneId.systemDefault();
//时区的日期和时间
ZonedDateTime zonedDateTime = localDateTime.atZone(zoneId);

//获取时刻
Date date = Date.from(zonedDateTime.toInstant());
System.out.println("格式化前：localDateTime:" + localDateTime + " Date:" + date);

//格式化LocalDateTime、Date

DateTimeFormatter localDateTimeFormat = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");

SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");

System.out.println("格式化后：localDateTime:" + localDateTimeFormat.format(localDateTime) + " Date:" + dateFormat.format(date));
```
