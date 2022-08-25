---
title: "Java深度克隆"
date: 2012-09-28T08:33:51+13:00
draft: false
tags: ["Deep clone", "Apache Commons", "Java"]
readingTime: 6
customSummary: 在实际开发中经常遇到属性克隆的问题，比如在表现层提交的Request DTO，可能需要在控制层被映射成多个Java Bean，再传递到逻辑层来进行相应的业务处理，那么如何才能简单而又快速的完成属性克隆呢？
---

## 背景  
&nbsp;  
&nbsp;
在实际开发中经常遇到属性克隆的问题，比如在表现层提交的Request DTO，可能需要在控制层被映射成多个Java Bean，再传递到逻辑层来进行相应的业务处理，那么如何才能简单而又快速的完成属性克隆呢？

对于仅仅包含简单属性的Java Bean来说，Apache Commons里的BeanUtils 是个不错的选择，不过前提是对应属性必须具备相同的属性名。但是对于复杂属性来说， 如果两个属性的类型不同，或者类型参数不同，使用BeanUtils的时候都要格外小心。

举个例子： InvoiceDTO和InvoiceBean里面都有两个属性amount和charges，类型定义如下：
```
InvoiceDTO  [ amount -> AmountDTO,  charges -> List<ChargeDTO>]
InvoiceBean [ amount ->AmountBean,  charges -> List<ChargeBean>]
```

这种情况下Apache BeanUtils无法正确克隆amount和charges，那我们应该怎么办呢？
  
&nbsp;

## 解决方案  
&nbsp;
### 序列化与反序列化
\
利用Jason做过渡，这种做法最优雅，也最简单，但是，多转换多IO， 性能不行。  
&nbsp;
### 扩展Apache BeanUtils
\
通过扩展，让它支持复杂的类型转换，而不仅仅是支持简单的类型转换（比如BigDecimal和String的转换）。

要想进行正确扩展，首先得弄清楚BeanUtils本身的结构和工作原理。反编译commons的代码之后，发现它之所以能够支持简单类型转换，是因为内嵌了一系列的类型转换器，这些转换器在初始化的时候与其对应的目标类型被一起缓存到一个fast hashmap中，之后每次在利用内省进行属性设置的时候，会先获取目标属性的类型所对应的类型转换器，转换之后再将结果其set到目标对象上。

了解了BeanUtils的工作原理之后，我们发现扩展的两个关键点：新的类型转换器的定义以及注册，类型转换的入口方法适配。
  
&nbsp;
### 新的类型转换器的定义以及注册：  
&nbsp;
1. 首先是定义： 本例子中需要转换的是带泛型参数的类型，可以抽象一个ParameterizedTypeConverter接口：

```
import org.apache.commons.beanutils.Converter;
 
public abstract interface ParameterizedTypeConverter extends Converter {
	public abstract Object convert(Class targetClass, Class targetGenericType,
			Object sourceObj);
}
```
2. 然后对于List类型定义一个实现类ListConverter：
```
import java.util.ArrayList; 
import java.util.List;
import org.apache.commons.beanutils.BeanUtils;
import org.apache.commons.beanutils.ConversionException;
import org.springframework.util.CollectionUtils;
 
@SuppressWarnings("all")
public final class ListConverter implements ParameterizedTypeConverter {
 
	@Override
	public Object convert(Class type, Object valuet) {
		return null;
	}
 
	@Override
	public Object convert(Class targetClass, Class destGenericType,
			Object origValue) {
 
		if (!(origValue instanceof List)) {
			throw new ConversionException("Invalid List value");
		}
 
		List origList = List.class.cast(origValue);
		if (!CollectionUtils.isEmpty(origList)) {
 
			Class origGenericType = origList.get(0).getClass();
			if (origGenericType.equals(destGenericType)) {
				return origValue;
			} else {
				List model = new ArrayList();
				try {
					for (Object singleValue : origList) {
						Object dest = destGenericType.newInstance();
						Object orig = origGenericType.cast(singleValue);
						BeanUtils.copyProperties(dest, orig);
						model.add(dest);
					}
				} catch (Exception e) {
					throw new ConversionException(e);
				}
				return model;
			}
		}
 
		return origValue;
	}
}
```

3. 定义完转换类之后就是注册了：
```
import java.util.List; 
 
import org.apache.commons.beanutils.ConvertUtilsBean;
 
public class ARPConvertUtilsBean extends ConvertUtilsBean {
 
	@Override
	public void deregister() {
		super.deregister();
		register(new ListConverter(), List.class);
	}
}
```  
&nbsp;
### 类型转换的入口方法适配  
&nbsp;  
&nbsp;
我们看到新增的转换器转换属性需要一些额外的参数，那么对应调用转换器的方法也得修改：

```
import java.beans.PropertyDescriptor; 
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.ParameterizedType;
 
import org.apache.commons.beanutils.BeanUtilsBean;
import org.apache.commons.beanutils.ContextClassLoaderLocal;
import org.apache.commons.beanutils.Converter;
import org.apache.commons.beanutils.DynaBean;
import org.apache.commons.beanutils.DynaClass;
import org.apache.commons.beanutils.DynaProperty;
import org.apache.commons.beanutils.PropertyUtilsBean;
 
public class ARPBeanUtilsBean extends BeanUtilsBean {
 
	public ARPBeanUtilsBean() {
		super(new ARPConvertUtilsBean(), new PropertyUtilsBean());
	}
 
	public static synchronized BeanUtilsBean getInstance() {
		return ((BeanUtilsBean) new ContextClassLoaderLocal() {
			// Creates the default instance used when the context classloader is
			// unavailable
			protected Object initialValue() {
				return new ARPBeanUtilsBean();
			}
		}.get());
	}
 
	@Override
	public void copyProperty(Object bean, String name, Object value)
			throws IllegalAccessException, InvocationTargetException {
 
		Object target = bean;
		int delim = name.lastIndexOf(46);
		if (delim >= 0) {
			try {
				target = getPropertyUtils().getProperty(bean,
						name.substring(0, delim));
			} catch (NoSuchMethodException e) {
				return;
			}
			name = name.substring(delim + 1);
		}
 
		String propName = null;
		Class type = null;
		int index = -1;
		String key = null;
 
		propName = name;
		int i = propName.indexOf(91);
		if (i >= 0) {
			int k = propName.indexOf(93);
			try {
				index = Integer.parseInt(propName.substring(i + 1, k));
			} catch (NumberFormatException e) {
			}
			propName = propName.substring(0, i);
		}
		int j = propName.indexOf(40);
		if (j >= 0) {
			int k = propName.indexOf(41);
			try {
				key = propName.substring(j + 1, k);
			} catch (IndexOutOfBoundsException e) {
			}
			propName = propName.substring(0, j);
		}
 
		if (target instanceof DynaBean) {
			DynaClass dynaClass = ((DynaBean) target).getDynaClass();
			DynaProperty dynaProperty = dynaClass.getDynaProperty(propName);
			if (dynaProperty == null) {
				return;
			}
			type = dynaProperty.getType();
		} else {
			PropertyDescriptor descriptor = null;
			try {
				descriptor = getPropertyUtils().getPropertyDescriptor(target,
						name);
 
				if (descriptor == null)
					return;
			} catch (NoSuchMethodException e) {
				return;
			}
			type = descriptor.getPropertyType();
			if (type == null) {
				return;
			}
		}
 
		if (index >= 0) {
			Converter converter = getConvertUtils().lookup(
					type.getComponentType());
			if (converter != null) {
				value = converter.convert(type, value);
			}
			try {
				getPropertyUtils().setIndexedProperty(target, propName, index,
						value);
			} catch (NoSuchMethodException e) {
				throw new InvocationTargetException(e, "Cannot set " + propName);
			}
		} else if (key != null) {
			try {
				getPropertyUtils().setMappedProperty(target, propName, key,
						value);
			} catch (NoSuchMethodException e) {
				throw new InvocationTargetException(e, "Cannot set " + propName);
			}
		} else {
			Converter converter = getConvertUtils().lookup(type);
			if (converter != null) {
				if (converter instanceof ParameterizedTypeConverter) {
					try {
						ParameterizedType targetActualType = (ParameterizedType) target
								.getClass().getDeclaredField(propName)
								.getGenericType();
						Class<?> targetActualClass = (Class<?>) targetActualType
								.getActualTypeArguments()[0];
						value = ((ParameterizedTypeConverter) converter)
								.convert(type, targetActualClass, value);
					} catch (Exception e) {
						return;
					}
 
				} else {
					try {
						value = converter.convert(type, value);
					} catch (Exception e) {
						return;
					}
				}
			}else {
				try {
					value = ARPBeanUtils.cloneBean(type, value);
				} catch (Exception e) {
					return;
				}
			}
 
			try {
				getPropertyUtils().setSimpleProperty(target, propName, value);
			} catch (NoSuchMethodException e) {
				throw new InvocationTargetException(e, "Cannot set " + propName);
			}
		}
	}
}
```
可以看到最后一大段else里面就是重构之后的代码，对于所获得的转换器做类型判断，不同的转换器调用不同的转换方法。
  
&nbsp;
### 对外封装  
&nbsp;
```
import org.apache.commons.beanutils.BeanUtils; 
import org.apache.commons.beanutils.BeanUtilsBean;
 
/**
 * @ClassName: ARPBeanUtils
 * @Description: An extension of Apache common BeanUtils, can do the conversion
 *               between POJOs with attributes of different generic type, like
 *               from: InvoiceDTO-List<ChargeDTO> to
 *               InvoiceBean-List<ChargeBean>
 * @author ZHUGA3
 * @date Sep 27, 2012 1:33:25 PM
 * 
 */
public final class ARPBeanUtils extends BeanUtils {
 
	private static BeanUtilsBean utilBean = ARPBeanUtilsBean.getInstance();
 
	public static void copyProperties(Object dest, Object orig) {
		try {
			utilBean.copyProperties(dest, orig);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
 
	/*
	 * a non-argument constructor must be supplied in destClass
	 */
	public static <T> T cloneBean(Class<T> destClass, Object orig) {
		try {
			T dest = destClass.newInstance();
			copyProperties(dest, orig);
			return dest;
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}
}
```
  
&nbsp;
## 总结
\
至此，大功告成，测试一下，能快速完成例子中的克隆，有多快？和第一种方法对比一下：同时转换1W次的时候，使用扩展后的 ARPBeanUtils 需要200ms，而通过Jason，即使是使用效率最高的Jackson，转换1W次也需要6s，高下立判啊  
&nbsp;  
&nbsp;
