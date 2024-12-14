

> MONTHS\_BETWEEN(DATE1, DATE2\) 用来计算两个日期的月份差。


最近接到一个迁移需求，把`Oracle SQL`接口迁移到新平台上，但新平台是采用`Java`计算的方式，所以我需求把SQL逻辑转成Java语言。


在遇到`MONTHS_BETWEEN`时，遇到一些奇怪的问题，在此记录一下。


### 情景再现


一开始，我的大致思路：先计算出两个日期的月份差，再拿开始日期加上月份差再与结束日期计算出日差，如果日差大于0，月份差\+1；日差小于0，则月份差\-1。



> 为什么不保留小数？
> 
> 
> 因为在SQL逻辑中使用到MONTHS\_BETWEEN都是用来计算近x个月、未来x个月这类数据，只需要判断是否大于或小于某个整数，所有这里取整是没有问题的（当时是这样想的）。



```


|  | package com.chen.util; |
| --- | --- |
|  |  |
|  | import lombok.extern.slf4j.Slf4j; |
|  | import org.apache.commons.lang3.StringUtils; |
|  |  |
|  | import java.text.ParsePosition; |
|  | import java.text.SimpleDateFormat; |
|  | import java.time.ZoneId; |
|  | import java.time.temporal.ChronoUnit; |
|  | import java.time.temporal.Temporal; |
|  | import java.util.Calendar; |
|  | import java.util.Date; |
|  | import java.util.Objects; |
|  |  |
|  | @Slf4j |
|  | public class DateUtil { |
|  |  |
|  | public static final SimpleDateFormat yyyyMMddDateFormat = new SimpleDateFormat("yyyyMMdd"); |
|  |  |
|  | public static Date strToDate(String str) { |
|  | if (StringUtils.isBlank(str)) { |
|  | return null; |
|  | } |
|  | return yyyyMMddDateFormat.parse(str, new ParsePosition(0)); |
|  | } |
|  |  |
|  | /** |
|  | * 计算两个日期差月份差 |
|  | * |
|  | * @param begDate 开始日期 |
|  | * @param endDate 结束日期 |
|  | * @return 月份差 |
|  | */ |
|  | public static Integer monthsBetween(Date begDate, Date endDate) { |
|  | try { |
|  | if (Objects.isNull(begDate) || Objects.isNull(endDate)) { |
|  | return null; |
|  | } |
|  | Temporal beg = begDate.toInstant().atZone(ZoneId.systemDefault()).toLocalDate(); |
|  | Temporal end = endDate.toInstant().atZone(ZoneId.systemDefault()).toLocalDate(); |
|  | int between = (int) ChronoUnit.MONTHS.between(beg, end); |
|  | Calendar calendar = Calendar.getInstance(); |
|  | calendar.setTime(begDate); |
|  | calendar.add(Calendar.MONTH, between); |
|  | Date begDateNew = calendar.getTime(); |
|  | Temporal begNew = begDateNew.toInstant().atZone(ZoneId.systemDefault()).toLocalDate(); |
|  | long dayDiff = ChronoUnit.DAYS.between(begNew, end); |
|  | if (dayDiff > 0) { |
|  | between += 1; |
|  | } else if (dayDiff < 0) { |
|  | between -= 1; |
|  | } |
|  | return between; |
|  | } catch (Exception e) { |
|  | log.warn("DateUtil monthsBetweenWithMon() Occurred Exception.", e); |
|  | return null; |
|  | } |
|  | } |
|  |  |
|  | public static void main(String[] args) { |
|  | System.out.printf("%-9s %-9s %-3s\n", "日期1", "日期2", "月份差"); |
|  | String date1 = "20240405", date2 = "20240807"; |
|  | Integer between = monthsBetween(strToDate(date1), strToDate(date2)); |
|  | System.out.printf("%-10s %-10s  %-3s\n", date1, date2, between); |
|  | } |
|  | } |


```

#### 结果与Oracle比对




| 开始日期 | 结束日期 | JAVA | ORACLE |
| --- | --- | --- | --- |
| 20240405 | 20240807 | 5 | 4\.06451612903226 |
| 20240715 | 20240102 | \-7 | \-6\.41935483870968 |
| 20231130 | 20240131 | 3 | 2 |
| 20240117 | 20231224 | \-1 | \-0\.774193548387097 |
| 20240229 | 20240529 | \-3 | \-3 |
| 20240229 | 20240530 | \-4 | \-3\.03225806451613 |
| 20240229 | 20240531 | \-4 | \-3 |
| 20240731 | 20240430 | \-3 | 3 |


#### 结果分析


自测与冒烟测试都没发现问题，正式测试时，发现当两个日期均是月末时，就会导致结果不正确（结果中的20231130与20240131）。


并且还发现Orcale的`MONTHS_BETWEEN`在处理月末时更是打破常规思维！比如`20240731`的近3个月应该是从`20240501`开始计算的；还有一种情况是当两个日期中有一个日期是2月末时，与大月比较29号、30号、31号时，29号与31号的月份差居然是相同的。


查了很多资料最后在[ORACLE 日期函数 MONTHS\_BETWEEN](https://github.com)文章中找到原因。



> MONTHS\_BETWEEN函数返回两个日期之间的月份数。如果两个日期月份内天数相同，或者都是某个月的最后一天，返回一个整数，否则，返回数值带小数，以每天1/31月来计算月中剩余天数。如果日期1比日期2小 ，返回值为负数。


### 问题解决


思路：
日差 \= 如果两个日期都是月末，日差为0，否则 (开始日期日 \- 结束日期日)
月差 \= (开始日期年份 \* 12 \+ 开始日期月份) \- (结束日期年份 \* 12 \+ 结束日期月份) \+ (日差 / 31\)



```


|  | package com.chen.util; |
| --- | --- |
|  |  |
|  | import lombok.extern.slf4j.Slf4j; |
|  | import org.apache.commons.lang3.StringUtils; |
|  |  |
|  | import java.math.BigDecimal; |
|  | import java.math.RoundingMode; |
|  | import java.text.ParsePosition; |
|  | import java.text.SimpleDateFormat; |
|  | import java.time.ZoneId; |
|  | import java.time.temporal.ChronoUnit; |
|  | import java.time.temporal.Temporal; |
|  | import java.util.Calendar; |
|  | import java.util.Date; |
|  | import java.util.Objects; |
|  |  |
|  | @Slf4j |
|  | public class DateUtil { |
|  |  |
|  | public static final SimpleDateFormat yyyyMMddDateFormat = new SimpleDateFormat("yyyyMMdd"); |
|  |  |
|  | public static Date strToDate(String str) { |
|  | if (StringUtils.isBlank(str)) { |
|  | return null; |
|  | } |
|  | return yyyyMMddDateFormat.parse(str, new ParsePosition(0)); |
|  | } |
|  |  |
|  | /** |
|  | * 判断日期是否是月末 |
|  | * @param date 日期 |
|  | * @return 是否月末 |
|  | */ |
|  | public static Boolean isEndOfMonth(Calendar date) { |
|  | if (Objects.isNull(date)) { |
|  | return false; |
|  | } |
|  | return date.get(Calendar.DAY_OF_MONTH) == date.getActualMaximum(Calendar.DAY_OF_MONTH); |
|  | } |
|  |  |
|  | /** |
|  | * 适配ORACLE数据库MONTHS_BETWEEN()计算结果 |
|  | * MONTHS_BETWEEN(startDate, endDate) |
|  | * |
|  | * @param startDate 开始时间 |
|  | * @param endDate   结果时间 |
|  | * @return 月份差 |
|  | */ |
|  | public static BigDecimal oracleMonthsBetween(Date startDate, Date endDate) { |
|  | Calendar startCalendar = Calendar.getInstance(); |
|  | startCalendar.setTime(startDate); |
|  | Calendar endCalendar = Calendar.getInstance(); |
|  | endCalendar.setTime(endDate); |
|  |  |
|  | int startYear = startCalendar.get(Calendar.YEAR); |
|  | int endYear = endCalendar.get(Calendar.YEAR); |
|  | int startMonth = startCalendar.get(Calendar.MONTH); |
|  | int endMonth = endCalendar.get(Calendar.MONTH); |
|  | int startDay = startCalendar.get(Calendar.DATE); |
|  | int endDay = endCalendar.get(Calendar.DATE); |
|  | // 月份差 |
|  | double result = (startYear * 12 + startMonth) - (endYear * 12 + endMonth); |
|  | // 小数月份 |
|  | double countDay; |
|  | // 如果是两个日期都是月末，就只处理月份；否则使用日差 / 31 算出小数月份 |
|  | if (isEndOfMonth(startCalendar) && isEndOfMonth(endCalendar)) { |
|  | countDay = 0; |
|  | } else { |
|  | countDay = (startDay - endDay) / 31d; |
|  | } |
|  | result += countDay; |
|  | // 返回并保留14位小数位 |
|  | return BigDecimal.valueOf(result) |
|  | .setScale(14, RoundingMode.HALF_UP) |
|  | .stripTrailingZeros(); |
|  | } |
|  |  |
|  | public static void main(String[] args) { |
|  | System.out.printf("%-9s %-9s %-3s\n", "日期1", "日期2", "月份差"); |
|  | String date1 = "20240405", date2 = "20240807"; |
|  | BigDecimal between = oracleMonthsBetween(strToDate(date1), strToDate(date2)); |
|  | System.out.printf("%-10s %-10s  %-3s\n", date1, date2, between.toPlainString()); |
|  | } |
|  | } |


```

#### 结果与Oracle比对




| 开始日期 | 结束日期 | JAVA | ORACLE |
| --- | --- | --- | --- |
| 20240405 | 20240807 | \-4\.06451612903226 | \-4\.06451612903226 |
| 20240423 | 20240614 | \-1\.70967741935484 | \-1\.70967741935484 |
| 20240229 | 20240529 | \-3 | \-3 |
| 20240229 | 20240530 | \-3\.03225806451613 | \-3\.03225806451613 |
| 20240229 | 20240531 | \-3 | \-3 |
| 20230228 | 20230528 | \-3 | \-3 |
| 20231130 | 20240131 | \-2 | \-2 |
| 20231130 | 20240201 | \-2\.06451612903226 | \-2\.06451612903226 |
| 20240731 | 20240430 | 3 | 3 |
| 20240731 | 20240429 | 3\.06451612903226 | 3\.06451612903226 |
| 20240430 | 20240731 | \-3 | \-3 |
| 20240114 | 20231010 | 3\.12903225806452 | 3\.12903225806452 |


 本博客参考[FlowerCloud机场](https://hushicha.org)。转载请注明出处！
