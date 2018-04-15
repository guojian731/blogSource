---
title: mybatis-xml文件解释
date: 2017-12-11 17:26:38
tags: mybatis
---

# mybatis将表的属性用resultMap表示
``` xml

<resultMap id="BaseResultMap" type="com.model.TMarine" >

<id column="ID" property="id" jdbcType="INTEGER" />

<result column="ADDRESS" property="address" jdbcType="VARCHAR" />

<result column="AREA_ID" property="areaId" jdbcType="INTEGER" />

<result column="CONTACT" property="contact" jdbcType="VARCHAR" />

<result column="CREATE_TIME" property="createTime" jdbcType="TIMESTAMP" />

<result column="FAX" property="fax" jdbcType="VARCHAR" />

<result column="FIRST_LETTER" property="firstLetter" jdbcType="VARCHAR" />

<result column="IS_DEL" property="isDel" jdbcType="INTEGER" />

<result column="IS_MODABLE" property="isModable" jdbcType="INTEGER" />

<result column="IS_ORGAN" property="isOrgan" jdbcType="INTEGER" />

<result column="LEVEL_ID" property="levelId" jdbcType="INTEGER" />

<result column="MARINE_CODE" property="marineCode" jdbcType="VARCHAR" />

<result column="MARINE_NAME" property="marineName" jdbcType="VARCHAR" />

<result column="PHONE" property="phone" jdbcType="VARCHAR" />

<result column="POSTCODE" property="postcode" jdbcType="VARCHAR" />

<result column="UPDATE_TIME" property="updateTime" jdbcType="TIMESTAMP" />

<result column="PID" property="pid" jdbcType="INTEGER" />

</resultMap>
```

resultMap中的id标签，作为唯一表示。

 

# 套用sql，常将表的属性分隔。
``` xml

<sql id="Base_Column_List" >

ID, ADDRESS, AREA_ID, CONTACT, CREATE_TIME, FAX, FIRST_LETTER, IS_DEL, IS_MODABLE,

IS_ORGAN, LEVEL_ID, MARINE_CODE, MARINE_NAME, PHONE, POSTCODE, UPDATE_TIME, PID

</sql>

<select id="selectByPrimaryKey" resultMap="BaseResultMap" parameterType="java.lang.Integer" >

select

<include refid="Base_Column_List" />

from t_marine

where ID = #{id,jdbcType=INTEGER}

</select>
``` 

# #{}，${} 的区别

#{} JDBC preparedstatement参数的占位符标志，可以防止sql注入

${} 仅仅是字符串替换，而且有sql注入的危险

prepareStatement会先初始化SQL，先把这个SQL提交到数据库中进行预处理，多次使用可提高效率。  

createStatement不会初始化，没有预处理，没次都是从0开始执行SQL

# <where></where>

有时查询有条件，有时没有条件，用where可以根据情况判断是否用where，并删除多余的and

# <set></set>

用<set></set>根据判断，可以设定update具体修改哪些字段，而不需要全部修改，比如用户表里有creatTime表示添加时间并且有值，但是在修改的表单提交没有将creatTime添加进去，这样整体update就会将createTime变成空。使用<set>标签还可以自动去掉修改最后的逗号。

# <trim></trim>

实例：
``` xml

<trim prefix="(" suffix=")" suffixOverrides=",">

 

</trim>
``` 

在语句之前加入“(”,在末尾加入")",并去掉最后一个逗号。

# resultMap 和 resultType 的区别

resultType和resultMap 均指返回类型，resultType是直接表示返回类型的，而resultMap则是对外部ResultMap的引用，但是resultType跟resultMap不能同时存在。

个人理解：当你的数据只是想单纯的返回一个map，用resultType="java.util.HashMap",如果想返回一个对象或者一个对象的列表，resultMap="BaseResultMap",BaseResultMap是一个映射，即表的属性。
