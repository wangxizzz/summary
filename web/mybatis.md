### 在mybatis中,把复杂查询改为多条简单查询,比如需要查询两张表的字段,那么就可以分开每张表进行查询,然后在service层把结果返回就行了,没必要封装一个dto来接受这两个字段一起返回.

###动态sql:  
1.**if语句**
```java
<if test="hotelName != null and hotelName !=''"> // hotelName表示接口的参数名
and hotel_name = #{hotelName}  // hotel_name表示数据库的列
</if>
```

2.**where**
```java
<where>  // where查询条件,还有就是消除拼接的第一个 and,保证拼接后sql的正确性.
<if test="city != null and city !=''">
and city = #{city}
</if>
<if test="hotelName != null and hotelName !=''">
and hotel_name = #{hotelName}
</if>
</where>
```

3.**set**
```java
<set>  // set,更新数据库,还有就是消除拼接后的最后一个逗号,保证正确性.
<if test="price != null and price > 0">
price = #{price},
</if>
<if test="levelEnum != null">
level = #{levelEnum},
</if>
</set>
```

4.**foreach:批量插入与多条件查询**
```java
<foreach collection="levelList" item="level" close=")"
open="(" separator=",">
#{level}    // 拼接之后就是  in (1,3,4)
</foreach>
```

5.**choose when otherwise:类似与java的switch语句**
```java
<choose>
<when test="hotelName != null and hotelName !=''"> // 当when满足了一个就不执行后面的判断了.
where hotel_name = #{hotelName}
</when>
<when test="city != null and city !=''">
where city = #{city}
</when>
<otherwise>
</otherwise>
</choose>
```

6.mybatis自定义typeHandler.处理枚举与数据库字段的对应  
- https://blog.csdn.net/fighterandknight/article/details/51520402 
- https://blog.csdn.net/soonfly/article/details/67640580

7.mapper转义:
<div class="table-box"><table border="1" width="800" cellspacing="0" cellpadding="1"><tbody><tr><td style="text-align:center;">&lt;</td>
<td style="text-align:center;">&lt;=</td>
<td style="text-align:center;">&gt;</td>
<td style="text-align:center;">&gt;=</td>
<td style="text-align:center;">&amp;</td>
<td style="text-align:center;">'</td>
<td style="text-align:center;">"</td>
</tr><tr><td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;lt;</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;lt;=</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;gt;</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;gt;=</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;amp;</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;apos;</span></p>
</td>
<td>
<p class="p1" style="text-align:center;"><span class="s1">&amp;quot;</span></p>
</td>
</tr></tbody></table></div>

========================================================  