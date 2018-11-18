---
title: Spring-源码阅读之Constant类
date: 2018-11-18 22:17:17
categories:  Spring
---

# Constant 类

Constant 类位于 `org.springframework.core` 包下, 这个类可以将指定类的 public static final 的常量通过反射的方式添加到一个 map 中, 并提供了一些便捷性的方法获取这些常量值, 其源码如下:

<!-- more -->

```java
public class Constants {
	/**
	 * 全类名, 例如 org.springframework.beans.factory.xml.XmlBeanDefinitionReader
	 */
	private final String className;

	/**
	 * 保存指定类的 public static final 类型的字段的值到 fieldCache 中, 常量
	 */
	private final Map<String, Object> fieldCache = new HashMap<>();

	public Constants(Class<?> clazz) {
		Assert.notNull(clazz, "Class must not be null");
		this.className = clazz.getName();
		Field[] fields = clazz.getFields();
		for (Field field : fields) {
			// 只保存 public static final 类型的字段到 cache 中
			if (ReflectionUtils.isPublicStaticFinal(field)) {
				String name = field.getName();
				try {
					Object value = field.get(null);
					this.fieldCache.put(name, value);
				}
				catch (IllegalAccessException ex) {
					// just leave this field and continue
				}
			}
		}
	}

	public final String getClassName() {
		return this.className;
	}

	// 获取 fieldCache 元素的数量
	public final int getSize() {
		return this.fieldCache.size();
	}

	protected final Map<String, Object> getFieldCache() {
		return this.fieldCache;
	}

	// 获取对应 Filed 的值, 并尝试将值转换成 Number 类型
	public Number asNumber(String code) throws ConstantException {
		Object obj = asObject(code);
		if (!(obj instanceof Number)) {
			throw new ConstantException(this.className, code, "not a Number");
		}
		return (Number) obj;
	}

	// 获取对应 Filed 的值, 并尝试将值转换成 String 类型
	public String asString(String code) throws ConstantException {
		return asObject(code).toString();
	}

	// 获取对应 Filed 的值
	public Object asObject(String code) throws ConstantException {
		Assert.notNull(code, "Code must not be null");
		String codeToUse = code.toUpperCase(Locale.ENGLISH);
		Object val = this.fieldCache.get(codeToUse);
		if (val == null) {
			throw new ConstantException(this.className, codeToUse, "not found");
		}
		return val;
	}

	// 返回所有以 namePrefix 为前缀的 Filed 的名称集合
	public Set<String> getNames(@Nullable String namePrefix) {
		String prefixToUse = (namePrefix != null ? namePrefix.trim().toUpperCase(Locale.ENGLISH) : "");
		Set<String> names = new HashSet<>();
		for (String code : this.fieldCache.keySet()) {
			if (code.startsWith(prefixToUse)) {
				names.add(code);
			}
		}
		return names;
	}

	// 返回所有以 propertyName 为前缀的 Filed 的名称集合
	public Set<String> getNamesForProperty(String propertyName) {
		return getNames(propertyToConstantNamePrefix(propertyName));
	}

	// 返回所有以 nameSuffix 为前缀的 Filed 的名称集合
	public Set<String> getNamesForSuffix(@Nullable String nameSuffix) {
		String suffixToUse = (nameSuffix != null ? nameSuffix.trim().toUpperCase(Locale.ENGLISH) : "");
		Set<String> names = new HashSet<>();
		for (String code : this.fieldCache.keySet()) {
			if (code.endsWith(suffixToUse)) {
				names.add(code);
			}
		}
		return names;
	}

	// 返回所有以 nameSuffix 为前缀的 Filed 的值集合
	public Set<Object> getValues(@Nullable String namePrefix) {
		String prefixToUse = (namePrefix != null ? namePrefix.trim().toUpperCase(Locale.ENGLISH) : "");
		Set<Object> values = new HashSet<>();
		this.fieldCache.forEach((code, value) -> {
			if (code.startsWith(prefixToUse)) {
				values.add(value);
			}
		});
		return values;
	}

	// 返回所有以 propertyName 为前缀的 Filed 的值集合
	public Set<Object> getValuesForProperty(String propertyName) {
		return getValues(propertyToConstantNamePrefix(propertyName));
	}

	// 返回所有以 nameSuffix 为后缀的 Filed 的值集合
	public Set<Object> getValuesForSuffix(@Nullable String nameSuffix) {
		String suffixToUse = (nameSuffix != null ? nameSuffix.trim().toUpperCase(Locale.ENGLISH) : "");
		Set<Object> values = new HashSet<>();
		this.fieldCache.forEach((code, value) -> {
			if (code.endsWith(suffixToUse)) {
				values.add(value);
			}
		});
		return values;
	}

	// Filed 的名称的前缀匹配 namePrefix, 并且值和 value 相同, 返回这个 Filed 的名称, 否则抛出 ConstantException 异常
	public String toCode(Object value, @Nullable String namePrefix) throws ConstantException {
		String prefixToUse = (namePrefix != null ? namePrefix.trim().toUpperCase(Locale.ENGLISH) : "");
		for (Map.Entry<String, Object> entry : this.fieldCache.entrySet()) {
			if (entry.getKey().startsWith(prefixToUse) && entry.getValue().equals(value)) {
				return entry.getKey();
			}
		}
		throw new ConstantException(this.className, prefixToUse, value);
	}

	// Filed 的名称的后缀匹配 nameSuffix, 并且值和 value 相同, 返回这个 Filed 的名称, 否则抛出 ConstantException 异常
	public String toCodeForSuffix(Object value, @Nullable String nameSuffix) throws ConstantException {
		String suffixToUse = (nameSuffix != null ? nameSuffix.trim().toUpperCase(Locale.ENGLISH) : "");
		for (Map.Entry<String, Object> entry : this.fieldCache.entrySet()) {
			if (entry.getKey().endsWith(suffixToUse) && entry.getValue().equals(value)) {
				return entry.getKey();
			}
		}
		throw new ConstantException(this.className, suffixToUse, value);
	}

	// Filed 的名称的前缀匹配 propertyName, 并且值和 value 相同, 返回这个 Filed 的名称, 否则抛出 ConstantException 异常
	public String toCodeForProperty(Object value, String propertyName) throws ConstantException {
		return toCode(value, propertyToConstantNamePrefix(propertyName));
	}

	/**
	 * 对属性名按照规定的格式进行格式化, 示例:
	 *
	 * "imageSize" -> "IMAGE_SIZE"
	 * "imagesize" -> "IMAGESIZE"
	 * "ImageSize" -> "_IMAGE_SIZE"
	 * "IMAGESIZE" -> "_I_M_A_G_E_S_I_Z_E"
	 */
	public String propertyToConstantNamePrefix(String propertyName) {
		StringBuilder parsedPrefix = new StringBuilder();
		for (int i = 0; i < propertyName.length(); i++) {
			char c = propertyName.charAt(i);
			if (Character.isUpperCase(c)) {
				parsedPrefix.append("_");
				parsedPrefix.append(c);
			}
			else {
				parsedPrefix.append(Character.toUpperCase(c));
			}
		}
		return parsedPrefix.toString();
	}


    @SuppressWarnings("serial")
    public static class ConstantException extends IllegalArgumentException {

        public ConstantException(String className, String field, String message) {
		super("Field '" + field + "' " + message + " in class [" + className + "]");
		}

		public ConstantException(String className, String namePrefix, Object value) {
			super("No '" + namePrefix + "' field with value '" + value + "' found in class [" + className + "]");
		}
	}
}
```