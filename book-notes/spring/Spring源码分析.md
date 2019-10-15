1.**spring中ids/names唯一**
```java
// 在解析xml文件时，解析到<bean>标签会执行
protected void checkNameUniqueness(String beanName, List<String> aliases, Element beanElement) {
		String foundName = null;

		if (StringUtils.hasText(beanName) && this.usedNames.contains(beanName)) {
			foundName = beanName;
		}
		if (foundName == null) {
			foundName = CollectionUtils.findFirstMatch(this.usedNames, aliases);
		}
		if (foundName != null) {
			error("Bean name '" + foundName + "' is already used in this <beans> element", beanElement);
		}

		this.usedNames.add(beanName);
		this.usedNames.addAll(aliases);
	}

// usedNames注释
/**
* Stores all used bean names so we can enforce uniqueness on a per
* beans-element basis. Duplicate bean ids/names may not exist within the
* same level of beans element nesting, but may be duplicated across levels(在不同的</beans>下).
*/
private final Set<String> usedNames = new HashSet<>();
```

**spring如何解决bean的循环依赖问题？源码分析：**
