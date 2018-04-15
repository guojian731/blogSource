---
title: java-利用注解完成字典表对应数据
date: 2017-12-11 17:33:56
tags: [java基础,注解]
---

# 设计思路：

**1、首先将字典表的数据以Map的形式进行初始化，key格式为type+"@"+code,value值为正常回显的值，如"ta_sex"+"@"+"1", "男"**

**2、建立DictAcc注解，code代表是数据库字段还是需要回显的标识，type存储类型如ta_sex**
``` java

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DictACC {
    public String code();
    public String type();
}
```

**3.建立一个类BV，code表示数据库存储的名字，name表示需要回显的名字**
``` java
public class BV {
    
  public final static String code="code";    
  
  public final static String name="name";
  
}
```

**4.在pojo中字段sex，再建立sex_n表示要回显的字段（_n必须对应）**
``` java

public class User {
    
    @DictACC(code = BV.code, type = "ta_orgType")
    public String orgType="1";
    
    
    @DictACC(code = BV.name, type = "ta_orgType")
    public String orgType_n;
    
    @DictACC(code = BV.code, type = "ta_sex")
    public String sex="1";
    
    
    @DictACC(code = BV.name, type = "ta_sex")
    public String sex_n;


    public String getOrgType() {
        return orgType;
    }


    public void setOrgType(String orgType) {
        this.orgType = orgType;
    }


    public String getOrgType_n() {
        return orgType_n;
    }


    public void setOrgType_n(String orgType_n) {
        this.orgType_n = orgType_n;
    }


    public String getSex() {
        return sex;
    }


    public void setSex(String sex) {
        this.sex = sex;
    }


    public String getSex_n() {
        return sex_n;
    }


    public void setSex_n(String sex_n) {
        this.sex_n = sex_n;
    }
    
    
}
```
**5.代码如下：map相当与数据库中取出来的数据库字典表**
``` java

public class MainTest {
    public static void main(String[] args) throws IllegalArgumentException, IllegalAccessException {
        
        HashMap<String, String> map=new HashMap<String, String>();
        
        map.put("ta_orgType"+"@"+"1", "部门1");
        map.put("ta_orgType"+"@"+"2", "部门2");
        map.put("ta_orgType"+"@"+"3", "部门3");
        
        map.put("ta_sex"+"@"+"1", "男");
        map.put("ta_sex"+"@"+"2", "女");
        
        User user=new User();
        User user1=new User();
        
        List<User> userList=new ArrayList<User>();
        userList.add(user);
        userList.add(user1);
        Doaction.fill(userList,map);
        for (int i = 0; i < userList.size(); i++) {
            System.out.println(userList.get(0).getOrgType_n());
        }
    }
}
```

**6.利用反射完成对sex_n等赋值**
``` java

public class Doaction {

    public static <T> void fill(List<T> pojoList, HashMap<String, String> dictmap)
            throws IllegalArgumentException, IllegalAccessException {

        Map<String, Map<String, Field>> map = getDictMap(pojoList.get(0));

        Map<String, Field> codeMap = map.get("codeMap");
        Map<String, Field> nameMap = map.get("nameMap");
        for (T pojo:pojoList) {
            fillname(pojo, codeMap, nameMap, dictmap);
        }
    }
    /**
     * 获取带DictACC字典注解的字段，并拼接成Map 
     * @param t
     * @return
     */
    public static <T> Map<String, Map<String, Field>> getDictMap(T t) {
        //字典code值组成的Map
        Map<String, Field> codeMap = new HashMap<String, Field>();
        //需要从code值转成value组成的Map
        Map<String, Field> nameMap = new HashMap<String, Field>();
        List<Field> fieldlist =new ArrayList<Field>();
        //获取父类的所有field
        fieldlist.addAll(Arrays.asList(t.getClass().getSuperclass().getDeclaredFields()));
        //获取本类的所有field
        fieldlist.addAll(Arrays.asList(t.getClass().getDeclaredFields()));
        for (Field field : fieldlist) {
            if (field.getAnnotation(DictACC.class) != null) {
                DictACC ann = field.getAnnotation(DictACC.class);
                if (ann.code().equals(BV.code)) {
                    codeMap.put(ann.type() + "@" + field.getName(), field);
                } else if (ann.code().equals(BV.name)) {
                    nameMap.put(ann.type() + "@" + field.getName(), field);
                }
            }
        }
        Map<String, Map<String, Field>> map = new HashMap<String, Map<String, Field>>();
        map.put("codeMap", codeMap);
        map.put("nameMap", nameMap);
        return map;

    }
    
    public static <T> void fillname(T t, Map<String, Field> codeMap,
            Map<String, Field> nameMap, HashMap<String, String> dictmap)
            throws IllegalArgumentException, IllegalAccessException {
        for (Entry<String, Field> entry : nameMap.entrySet()) {
            //获取codekey
            String codeKey = entry.getKey().substring(0,
                    entry.getKey().length() - 2);
            if (codeMap.get(codeKey) != null) {
                Field field = codeMap.get(codeKey);
                //获取code的值
                String value = String.valueOf(field.get(t));
                String targetKey = codeKey.split("@")[0] + "@" + value;
                if (dictmap.containsKey(targetKey)) {
                    //从字典表获取那么中应该存的值
                    String targetValue = dictmap.get(targetKey);
                    Field targetField = entry.getValue();
                    targetField.setAccessible(true);
                    //给name赋值
                    targetField.set(t, targetValue);
                }

            }
        }
    }
}
```