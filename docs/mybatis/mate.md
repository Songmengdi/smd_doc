# MateClass解析
```java
// 这个类是用来表示类的元信息
public class MetaClass {

  private final ReflectorFactory reflectorFactory;
  // 存放解析后的 某个类的元信息的 (属性,set,get方法,构造函数等) 
  private final Reflector reflector;

  private MetaClass(Class<?> type, ReflectorFactory reflectorFactory) {
    this.reflectorFactory = reflectorFactory;
    this.reflector = reflectorFactory.findForClass(type);
  }

  // MateClass 构造方法私有 ,拥有两个成员变量 反射器工厂类, 反射器,通过使用forClass来创建类
  // mybatis实现的ReflectorFactory 只有 DefaultReflectorFactory 
  public static MetaClass forClass(Class<?> type, ReflectorFactory reflectorFactory) {
    return new MetaClass(type, reflectorFactory);
  }
  // ...hasSetter
  // ...hasGetter
  // ...hasDefaultConstructor
  // ...getGetterName
  // ...getSetterName
  
}

// 反射器工厂类接口
public interface ReflectorFactory {

  // 当前类是否已经缓存了
  boolean isClassCacheEnabled();
  // 设置是否使用缓存
  void setClassCacheEnabled(boolean classCacheEnabled);
  // 解析类的Class获取类的元信息
  Reflector findForClass(Class<?> type);
}

// 保存和解析类的元信息
public class Reflector {
  // 类的类型
  private final Class<?> type;
  // 所有可读属性的属性名数组
  private final String[] readablePropertyNames;
  // 所有科协属性的属性名数组
  private final String[] writablePropertyNames;
  // set方法
  private final Map<String, Invoker> setMethods = new HashMap<>();
  // get方法
  private final Map<String, Invoker> getMethods = new HashMap<>();
  // setType name: type
  private final Map<String, Class<?>> setTypes = new HashMap<>();
  // getType name: type
  private final Map<String, Class<?>> getTypes = new HashMap<>();
  // 默认构造
  private Constructor<?> defaultConstructor;
  // 不区分大小写的属性集
  private Map<String, String> caseInsensitivePropertyMap = new HashMap<>();

  // 构造函数
  public Reflector(Class<?> clazz) {
    type = clazz;
    // 解析默认构造
    addDefaultConstructor(clazz);
    // 解析get方法
    addGetMethods(clazz);
    // 解析set方法
    addSetMethods(clazz);
    // 解析字段
    addFields(clazz);
    // 设置可读属性名
    readablePropertyNames = getMethods.keySet().toArray(new String[0]);
    // 设置可写属性名
    writablePropertyNames = setMethods.keySet().toArray(new String[0]);
    // 属性名大写 : 属性名 el: NAME: name
    for (String propName : readablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
    for (String propName : writablePropertyNames) {
      caseInsensitivePropertyMap.put(propName.toUpperCase(Locale.ENGLISH), propName);
    }
  }


  
}
```

### 构造方法解析
```java
public class Reflector {
  // ...省略代码
  private void addDefaultConstructor(Class<?> clazz) {
    Constructor<?>[] constructors = clazz.getDeclaredConstructors();
    // 查找构造方法中无参构造,并赋值给defaultConstructor
    Arrays.stream(constructors).filter(constructor -> constructor.getParameterTypes().length == 0)
      .findAny().ifPresent(constructor -> this.defaultConstructor = constructor);
  }
  
}
```
类中如果有无参构造则使用这个无参构造

### 类中get方法解析
```java
public class Reflector {
  
  private void addGetMethods(Class<?> clazz) {
    Map<String, List<Method>> conflictingGetters = new HashMap<>();
    // 获取类的所有方法
    Method[] methods = getClassMethods(clazz);
    // 过滤出无参的get方法 并且方法名命名符合 isXXX getXXX
    Arrays.stream(methods).filter(m -> m.getParameterTypes().length == 0 && PropertyNamer.isGetter(m.getName()))
      // 将重名的方法放到conflictingGetters中
      .forEach(m -> addMethodConflict(conflictingGetters, PropertyNamer.methodToProperty(m.getName()), m));
    // 解决重名get方法 (比如有 isName() getName() 两个方法,使用哪个作为name的get方法? 
    resolveGetterConflicts(conflictingGetters);
  }
  // 将有争议的方法加入到conflictingMethods中
  private void addMethodConflict(Map<String, List<Method>> conflictingMethods, String name, Method method) {
    //  方法名称符合 isXxx getXxx setXxx 的通过 key value[]的形式存储
    if (isValidPropertyName(name)) {
      List<Method> list = conflictingMethods.computeIfAbsent(name, k -> new ArrayList<>());
      list.add(method);
    }
  }
  //解决get方法的冲突 如果name只对应一个方法则直接使用addGetMethod添加
  private void resolveGetterConflicts(Map<String, List<Method>> conflictingGetters) {
    for (Entry<String, List<Method>> entry : conflictingGetters.entrySet()) {
      Method winner = null;
      String propName = entry.getKey();
      boolean isAmbiguous = false;
      for (Method candidate : entry.getValue()) {
        if (winner == null) {
          winner = candidate;
          continue;
        }
        // 获胜者
        Class<?> winnerType = winner.getReturnType();
        // 候选人
        Class<?> candidateType = candidate.getReturnType();
        // 两者的返回类型一致
        if (candidateType.equals(winnerType)) {
          if (!boolean.class.equals(candidateType)) {
            // 如果返回类型不是 boolean  eg  String isName(); String getName(); 无法分辨 直接返回
            isAmbiguous = true;
            break;
          } else if (candidate.getName().startsWith("is")) {
            // 如果是boolean返回则使用is开头的get方法作为 属性的get方法
            winner = candidate;
          }
        } else if (candidateType.isAssignableFrom(winnerType)) {
          // 候选类型是 当前选中类型的超类,则使用子类作为get方法
          // OK getter type is descendant
        } else if (winnerType.isAssignableFrom(candidateType)) {
          // 反正反过来
          winner = candidate;

          // 总结就是 如果其中一个的返回结果是另一个的子类,使用子类返回的get方法作为属性的get方法
        } else {
          // 两个返回结果没有关系, 无法分辨直接中断
          isAmbiguous = true;
          break;
        }
      }
      addGetMethod(propName, winner, isAmbiguous);
    }
  }
  
  // 添加方法
  private void addGetMethod(String name, Method method, boolean isAmbiguous) {
    // 如果不可分辨,则设置当前属性的get方法是一个有纷争的方法
    MethodInvoker invoker = isAmbiguous
        ? new AmbiguousMethodInvoker(method, MessageFormat.format(
            "Illegal overloaded getter method with ambiguous type for property ''{0}'' in class ''{1}''. This breaks the JavaBeans specification and can cause unpredictable results.",
            name, method.getDeclaringClass().getName()))
        : new MethodInvoker(method);
    // 将属性名 和 方法名存储
    getMethods.put(name, invoker);
    Type returnType = TypeParameterResolver.resolveReturnType(method, type);
    // 将属性名 和属性的类型 存储
    getTypes.put(name, typeToClass(returnType));
  }


}
```
> 解析过程
1. 寻找类中的所有get方法(符合 isXxx getXXX setXxx 的形式的方法)
2. 如果get方法的属性名对应的方法有多个
    1. 如果两个方法同类型 且都是boolean返回则使用is开头的方法
    2. 如果两个方法的返回类型其中一个是另一个的子类,则使用子类返回值的get方法
    3. 对于其他的无法解析抛出运行时异常
    4. 将属性名: 方法名 存储到 getMethods
    5. 将属性名: 类型 存储到 getTypes中   

### 类中set方法解析
```java
public class Reflector {
  private void addSetMethods(Class<?> clazz) {
    Map<String, List<Method>> conflictingSetters = new HashMap<>();
    Method[] methods = getClassMethods(clazz);
    Arrays.stream(methods).filter(m -> m.getParameterTypes().length == 1 && PropertyNamer.isSetter(m.getName()))
      .forEach(m -> addMethodConflict(conflictingSetters, PropertyNamer.methodToProperty(m.getName()), m));
    resolveSetterConflicts(conflictingSetters);
  }
  // set方法纠纷解决
  private void resolveSetterConflicts(Map<String, List<Method>> conflictingSetters) {
    for (Entry<String, List<Method>> entry : conflictingSetters.entrySet()) {
      String propName = entry.getKey();
      List<Method> setters = entry.getValue();
      Class<?> getterType = getTypes.get(propName);
      // get方法是否存在纷争
      boolean isGetterAmbiguous = getMethods.get(propName) instanceof AmbiguousMethodInvoker;
      // set方法是否存在纷争
      boolean isSetterAmbiguous = false;
      Method match = null;
      for (Method setter : setters) {
        // get方法没有纷争, set方法的第一个参数的类型是get的返回类型,则认为是最佳匹配了
        if (!isGetterAmbiguous && setter.getParameterTypes()[0].equals(getterType)) {
          // should be the best match
          match = setter;
          break;
        }
        // set没有纷争
        if (!isSetterAmbiguous) {
          
          match = pickBetterSetter(match, setter, propName);
            = match == null;
        }
      }
      if (match != null) {
        addSetMethod(propName, match);
      }
    }
  }  
  
  // 挑选最佳的set方法
  private Method pickBetterSetter(Method setter1, Method setter2, String property) {
    // 参数1没有值则使用传入值(第一次进入循环)
    if (setter1 == null) {
      return setter2;
    }
    // 方法1 的arg[0]
    Class<?> paramType1 = setter1.getParameterTypes()[0];
    // 方法2 的arg[0]
    Class<?> paramType2 = setter2.getParameterTypes()[0];
    // 参数其中一个是另一个子类,使用超类(接口)使用超类(接口)的入参项作为set方法
    if (paramType1.isAssignableFrom(paramType2)) {
      return setter2;
    } else if (paramType2.isAssignableFrom(paramType1)) {
      return setter1;
    }
    // 两个方法无法分辨
    MethodInvoker invoker = new AmbiguousMethodInvoker(setter1,
        MessageFormat.format(
            "Ambiguous setters defined for property ''{0}'' in class ''{1}'' with types ''{2}'' and ''{3}''.",
            property, setter2.getDeclaringClass().getName(), paramType1.getName(), paramType2.getName()));
    // setMethods中存入纷争属性方法
    setMethods.put(property, invoker);
    Type[] paramTypes = TypeParameterResolver.resolveParamTypes(setter1, type);
    // 存入属性值和纷争的属性名和类型
    setTypes.put(property, typeToClass(paramTypes[0]));
    return null;
  }
}

```
####  解析字段
> 1. 字段解析将不再set和get方法中解析出来的属性(不包括final和static的顺序)存入,并自动设置set和get方法并存入
> 2. 对于该类的超类,继续向上迭代存入