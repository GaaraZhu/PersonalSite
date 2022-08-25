---
title: "基于Aviator的注解驱动验证框架"
date: 2012-08-25T09:20:14+13:00
draft: false
tags: ["Data quality", "Declarative validation", "Aviator", "Java"]
readingTime: 8
customSummary: 程序开发过程中，在同一系统中层层之间数据传递或者是异构系统之间同步异步通信的时候，我们经常需要对Java Bean进行属性验证，来决定是否继续后续process，或者直接抛出error message。EasyValidation是基于Aviator DIY的申明式验证框架，能解决传统解决方案存在的弊端与不足。
---
## 背景
\
程序开发过程中，在同一系统中层层之间数据传递或者是异构系统之间同步异步通信的时候，我们经常需要对Java Bean进行属性验证，来决定是否继续后续process，或者直接抛出error message。
  
&nbsp;
## 传统解决方案
\
传统的做法，给每个验证场景加一个验证类，专门负责所有属性的验证，以及验证结果的校验和处理，这种做法最直白，但是不够灵活。第二种就是Java 6自带的验证框架 Bean validation, 用注解的形式来表达属性对应的约束，Bean validation在很大程度上提高了数据验证的灵活性和复用性，但是第一，个人感觉扩展比较麻烦，首先你需要自定义个注解来代表对应的约束，其次你必须新建一个验证器，最后你必须在注解类上做好两者的mapping，对于每种约束，都逃不开这三步；第二点，Bean validation只能完成bean中单一属性的验证，不支持跨属性验证。
  
&nbsp;
### EasyValidation!
\
第三种做法就是基于Aviator DIY的验证框架EasyValidation，Aviator是淘宝开发的一款java表达式求值工具，感兴趣的朋友可以google一下。为了灵活性，和Bean validation一样，EasyValidation也是注解驱动的。我们认为所谓的约束其实就是一个结果为 true 或者 false 的表达式，所以首先把这些约束表达式加在属性上面作为元数据，然后通过反射去获取表达式，最后利用aviator去求值，再对结果做处理。

首先构建约束表达式的注解：

```JAVA
@Target(FIELD)
@Retention(RUNTIME)
public @interface Validation {
    public abstract String condition();
    public abstract String errorMsg();
}
```

单个属性支持多约束验证：
```JAVA
@Target(FIELD)@Retention(RUNTIME)
public @interface Validations {
    public abstract Validation[] values();
}
```

由于Aviator计算表达式之前会利用asm将表达式生成java字节码，并且支持cache，所以定义一个annotaion，加在javabean上可以表示是否需要对其属性约束做cache:

```JAVA
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExpressionCacheable {}
```

核心验证方法：

```JAVA
public static List < String > plainValidate(Object pojo, boolean validateAll) {

    BeanUtilBeanAccess access = new BeanUtilBeanAccess();
    List < Field > targetFields = new ArrayList < Field > ();
    List < String > msgList = new ArrayList < String > ();
    Map < String, Object > env = new HashMap < String, Object > ();
    boolean needCacheExpression = pojo.getClass().isAnnotationPresent(
        ExpressionCacheable.class);

    for (Field field: pojo.getClass().getDeclaredFields()) {
        try {
            String fieldName = field.getName();
            Object value = access.get(fieldName, pojo);
            env.put(fieldName, value);

            if (field.isAnnotationPresent(Validations.class) ||
                field.isAnnotationPresent(Validation.class)) {
                targetFields.add(field);
            }
        } catch (Exception e) {
            continue;
        }
    }

    for (Field field: targetFields) {
        for (Validation validation: getValidateAnnotations(field)) {
            boolean validationPassed = true;

            String expression = validation.condition();
            Expression compiledExp = AviatorEvaluator.compile(expression,
                needCacheExpression);
            try {
                validationPassed = (Boolean) compiledExp.execute(env);
            } catch (Exception e) {
                validationPassed = false;
            }
            if (!validationPassed) {
                String errorMsg = validation.errorMsg();
                if (errorMsg.contains("#".concat(field.getName()))) {
                    Object value = env.get(field.getName());
                    String parm = (null == value) ? null : value.toString();
                    errorMsg = errorMsg.replaceAll(
                        "#".concat(field.getName()), parm);
                }
                msgList.add(errorMsg);


                if (!validateAll) {
                    return msgList;
                } else {
                    continue;
                }
            }
        }
    }

    return msgList;
}
```

大家可以看到，第一步会获得java bean属性名和属性值，并且放到一个map中，这就是aviator的运行环境，EasyValidation之所以能支持跨属性验证，主要就是因为有这个env map，在约束表达式里面可以直接使用属性名，代表该属性的值，aviator计算表达式的时候会把属性名换成env map中对应的键值来进行计算。
  
&nbsp;
### 流程图

​![workflow](/images/aviator-validator/workflow.png)

Aviator除了支持基本的运算表达式之外，还内嵌了一些常用的函数，比如string.contains, string.endswiths 等等，你可以直接在约束表达式里使用他们.
  
&nbsp;
## 自定义函数
\
看到这里，可能你会问，如何自定义函数，很简单！两步！
比如我们想加一个函数，来判断属性值是否是一个合法的数值：  
&nbsp;
### 第一步，定义函数

```JAVA
@Function
public class NumberValidFunction extends AbstractFunction {

    @Override
    public String getName() {
        return "number.valid";
    }


    public AviatorObject call(Map < String, Object > env, AviatorObject arg1) {
        String targetStr = FunctionUtils.getStringValue(arg1, env);
        return AviatorBoolean.valueOf(CalculationUtils.isNumberic(targetStr));
    }
}
```

直接扩展Aviator中的AbstractFunction，覆盖getName()方法和call(...)方法，其中，name就是表达式中使用的函数名。  
&nbsp;
### 第二步，注册函数：  
&nbsp;
```JAVA
AviatorEvaluator.addFunction(new NumberValidFunction());
```  
&nbsp;
## Demo
\
这样，该函数就可以使用了！好了，准备活动做好了，让我们来看一个完整的例子吧，对Student对象我们有一些验证，于是我们先把约束表达式加在属性上面：

```JAVA
@ExpressionCacheable
public class Student{
    @Validation(condition = "string.notEmpty(name) && string.startsWith('gaara')", 
                                errorMsg = "Invalid name: #name")
    private String name;
 
    @Validation(condition = "age < fatherAge", errorMsg = "Invalid age")
    private int age;
    private int fatherAge;
 
     public int getAge(){
         return age;
     }
 
    public void setAge(int age){
        this.age = age;
     }
 
    public int getFatherAge(){
        return fatherAge;
     }
 
     public void setFatherAge(int fatherAge){
          this.fatherAge = fatherAge;
     }
 
    public String getName(){
       return name;
    }
 
 
    public void setName(String name){
       this.name = name;
    }
 
 
}
```

然后我们对一个Student实例进行验证：
```JAVA
List<String> errorMsgs = ValidationUtils.plainValidate(student, true);
```
\
一句话，就得到所有的errorMsg(第二个参数如果为false，返回第一个error message，不再继续验证)。
  
&nbsp;
## 总结  
&nbsp;
* 不要滥用cache。
* 自定义函数的注册可以通过注解+扫描器在系统初始化的时候完成。
* EasyValidation中所有的验证约束都是放在java类上，不支持XSD level的验证约束，所以不支持EMS，因为EMS所需的 jar 一般都是用第三方控件通过XSD去生成，虽然你可以选择生成源码再把约束append上去，但是每次更新schema都会让你丢失原有的约束。
* 性能方面测试下来没什么问题，如果实在对反射不放心的话，可以试试别的方法，比如Unsafe类，但是前提是关掉编译器对Restricted API的Errors/Warnings。  
&nbsp;  
&nbsp;
