# ButterKnife 研究
------------------------------

ButterKnife工作分为三部分，

1. 搜索代码中被```@Bind```注解标记的元素(feild或method)，将注解中的id号与元素建立对应关系
2. 动态生成绑定相关的代码
3. 在用户调用的```bing```方法中执行最后的绑定

前两布在预处理过程中执行，最后一步在用户代码中执行。

预处理过程的入口在`butterknife.compiler.ButterKnifeProcessor`这个类的`public boolean process(Set<? extends TypeElement> elements, RoundEnvironment env)`方法。

`butterknife.compiler.ButterKnifeProcessor`继承了jdk中的`javax.annotation.processing.AbstractProcessor`。

`Set<String> ButterKnifeProcessor.getSupportedAnnotationTypes()`怎么理解，该函数返回需要当前抽象处理器处理的注解列表。只有工程代码中出现了此列表中注册的注解，才会调用后续的`process(Set<? extends TypeElement> elements, RoundEnvironment env)`方法。而`process(Set<? extends TypeElement> elements, RoundEnvironment env)`方法中处理的注解和`Set<String> ButterKnifeProcessor.getSupportedAnnotationTypes()`返回的注解列表并没有直接的包含和被包含关系。
比如，我们可以让`getSupportedAnnotationTypes()`返回的Set中只包含`Unbinder`，只要Sample项目中有使用到`@Unbinder`注解的元素，之后调用的`process(Set<? extends TypeElement> elements, RoundEnvironment env)`依然可以扫描出`@Bind`注解的元素，并进行绑定。
但是如果`getSupportedAnnotationTypes()`返回的Set中的注解在Sample项目中并没有被使用到，比如我们只返回`BindArray`、 `BindBitmap`、 `BindBool`、 `BindColor`、 `BindDimen`、 `BindDrawable`、 `BindInt`、 `BindString`这几个，那么处理器就不会执行`process(Set<? extends TypeElement> elements, RoundEnvironment env)`这个方法，绑定操作就会失效，应用启动立即core dump。


解析Sample工程中的注解，位于`ButterKnifeProcessor.findAndParseTargets(RoundEnvironment env)`中，该方法由`ButterKnifeProcessor.process(Set<? extends TypeElement> elements, RoundEnvironment env)`调用。

以最基本的`@Bind`注解为例，调用`env.getElementsAnnotatedWith(Bind.class)`可以立刻获得Sample工程中的所有被`@Bind`注解的元素。
```
elementsWithBind = {LinkedHashSet@9711}  size = 9
 0 = {Symbol$VarSymbol@9714} "title"
  name = {UnsharedNameTable$NameImpl@9732} "title"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 1 = {Symbol$VarSymbol@9715} "subtitle"
  name = {UnsharedNameTable$NameImpl@9739} "subtitle"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 2 = {Symbol$VarSymbol@9716} "hello"
  name = {UnsharedNameTable$NameImpl@9744} "hello"
  type = {Type$ClassType@9745} "android.widget.Button"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 3 = {Symbol$VarSymbol@9717} "listOfThings"
  name = {UnsharedNameTable$NameImpl@9750} "listOfThings"
  type = {Type$ClassType@9751} "android.widget.ListView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 4 = {Symbol$VarSymbol@9718} "footer"
  name = {UnsharedNameTable$NameImpl@9756} "footer"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 5 = {Symbol$VarSymbol@9719} "headerViews"
  name = {UnsharedNameTable$NameImpl@9761} "headerViews"
  type = {Type$ClassType@9762} "java.util.List<android.view.View>"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
 6 = {Symbol$VarSymbol@9720} "word"
  name = {UnsharedNameTable$NameImpl@9767} "word"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
 7 = {Symbol$VarSymbol@9721} "length"
  name = {UnsharedNameTable$NameImpl@9773} "length"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
 8 = {Symbol$VarSymbol@9722} "position"
  name = {UnsharedNameTable$NameImpl@9778} "position"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
```
该集合中的元素属性主要包括元素的名字，类型，所属的类。

之后再一个foreach循环中对集合中的每一个`element`执行`parseBind(element, targetClassMap, erasedTargetNames)`

`targetClassMap`里面会保存`@Bind`注解中的value，（即布局文件中的view id）和被注解的元素的对应关系。这里面也会包含其他的注解。

在`parseBind(Element element, Map<TypeElement, BindingClass> targetClassMap, Set<TypeElement> erasedTargetNames)`当中，经过一系列的验证排除掉对集合类元素的注解这种异常场景之后，最终会调用`parseBindOne(element, targetClassMap, erasedTargetNames)`

而在`parseBindOne(Element element, Map<TypeElement, BindingClass> targetClassMap, Set<TypeElement> erasedTargetNames)`中，又要对被注解的元素`element`是否是安卓View的子类实例以及注解的value是否只有一个值进行校验。也就是是说，通过`@Bind`绑定的只能是`View`，每一个`View`只能绑定一个id。

校验结束后，通过`element`所属的类`enclosingElement`（通过`element.getEnclosingElement()`获取）为key到`targetClassMap`中查找`bindingClass`，对第一个`element`进行解析的时候肯定是查找不到的，于是就会在`getOrCreateTargetClass(targetClassMap, enclosingElement)`中创建一个`BindingClass`类型的`bindingClass`变量，并且以`enclosingElement`为key添加到`targetClassMap`中，后续的`element`解析过程中会复用这个`bindingClass`变量。

`BindingClass`的精髓在于它的`Map<Integer, ViewBindings> viewIdMap`成员变量，这个`Map`变量保存了布局中的view id和`Activity`中被`@Bind`注解的`View`类型及其子类型成员变量的对应关系。
成员变量信息保存在`ViewBindings`的子类`FieldViewBinding`中。`FieldViewBinding`的实例通过`parseBindOne`方法中的最后几行代码

```java
String name = element.getSimpleName().toString();
TypeName type = TypeName.get(elementType);
boolean required = isFieldRequired(element);

FieldViewBinding binding = new FieldViewBinding(name, type, required);

```
进行构建，最后通过`bindingClass.addField(id, binding);`添加到`viewIdMap`当中。

以下为`@Bind`注解解析完成后`targetClassMap`的快照（篇幅有限，删除了一些无关信息）
```
targetClassMap = {LinkedHashMap@9687}  size = 2
+-0 = {LinkedHashMap$Entry@9751}
| +-key = {Symbol$ClassSymbol@9753} "com.example.butterknife.SimpleActivity"
| +-value = {BindingClass@9754}
|   +-viewIdMap = {LinkedHashMap@9760}  size = 5
|   | +-0 = {LinkedHashMap$Entry@9770} "2130968576" ->
|   | |  key = {Integer@9775} "2130968576"
|   | |  value = {ViewBindings@9776}
|   | |   id = 2130968576
|   | |   fieldBindings = {LinkedHashSet@9791}  size = 1
|   | |    0 = {FieldViewBinding@9794}
|   | |     name = {String@9795} "title"
|   | |     type = {ClassName@9796} "android.widget.TextView"
|   | +-1 = {LinkedHashMap$Entry@9771} "2130968577" ->
|   | |  key = {Integer@9777} "2130968577"
|   | |  value = {ViewBindings@9778}
|   | |   id = 2130968577
|   | |   fieldBindings = {LinkedHashSet@9800}  size = 1
|   | |    0 = {FieldViewBinding@9803}
|   | |     name = {String@9804} "subtitle"
|   | |     type = {ClassName@9805} "android.widget.TextView"
|   | +-2 = {LinkedHashMap$Entry@9772} "2130968578" ->
|   | |  key = {Integer@9779} "2130968578"
|   | |  value = {ViewBindings@9780}
|   | |   id = 2130968578
|   | |   fieldBindings = {LinkedHashSet@9808}  size = 1
|   | |    0 = {FieldViewBinding@9811}
|   | |     name = {String@9812} "hello"
|   | |     type = {ClassName@9813} "android.widget.Button"
|   | +-3 = {LinkedHashMap$Entry@9773} "2130968579" ->
|   | |  key = {Integer@9781} "2130968579"
|   | |  value = {ViewBindings@9782}
|   | |   id = 2130968579
|   | |   fieldBindings = {LinkedHashSet@9816}  size = 1
|   | |    0 = {FieldViewBinding@9819}
|   | |     name = {String@9820} "listOfThings"
|   | |     type = {ClassName@9821} "android.widget.ListView"
|   | +-4 = {LinkedHashMap$Entry@9774} "2130968580" ->
|   |    key = {Integer@9783} "2130968580"
|   |    value = {ViewBindings@9784}
|   |     id = 2130968580
|   |     fieldBindings = {LinkedHashSet@9824}  size = 1
|   |      0 = {FieldViewBinding@9827}
|   |       name = {String@9828} "footer"
|   |       type = {ClassName@9829} "android.widget.TextView"
|   +-classPackage = {String@9765} "com.example.butterknife"
|   +-className = {String@9766} "SimpleActivity$$ViewBinder"   
+-1 = {LinkedHashMap$Entry@9752}
  +-key = {Symbol$ClassSymbol@9755} "com.example.butterknife.SimpleAdapter.ViewHolder"
  +-value = {BindingClass@9756}
    +-viewIdMap = {LinkedHashMap@9832}  size = 3
    | +-0 = {LinkedHashMap$Entry@9842} "2130968581" ->
    | |  key = {Integer@9845} "2130968581"
    | |  value = {ViewBindings@9846}
    | |   id = 2130968581
    | |   fieldBindings = {LinkedHashSet@9855}  size = 1
    | |    0 = {FieldViewBinding@9858}
    | |     name = {String@9859} "word"
    | |     type = {ClassName@9860} "android.widget.TextView"
    | +-1 = {LinkedHashMap$Entry@9843} "2130968582" ->
    | |  key = {Integer@9847} "2130968582"
    | |  value = {ViewBindings@9848}
    | |   id = 2130968582
    | |   fieldBindings = {LinkedHashSet@9863}  size = 1
    | |    0 = {FieldViewBinding@9866}
    | |     name = {String@9867} "length"
    | |     type = {ClassName@9868} "android.widget.TextView"
    | +-2 = {LinkedHashMap$Entry@9844} "2130968583" ->
    |    key = {Integer@9849} "2130968583"
    |    value = {ViewBindings@9850}
    |     id = 2130968583
    |     fieldBindings = {LinkedHashSet@9871}  size = 1
    |      0 = {FieldViewBinding@9874}
    |       name = {String@9875} "position"
    |       type = {ClassName@9876} "android.widget.TextView"
    +-classPackage = {String@9837} "com.example.butterknife"
    +-className = {String@9838} "SimpleAdapter$ViewHolder$$ViewBinder"   
```

我们可以从`targetClassMap`中读出诸如此类的以下信息：
```
有两个类中存在@Bind注解
  第一个类为com.example.butterknife.SimpleActivity
    该类中有5个被@Bind注解的成员
    第一个成员名字是title，类型是android.widget.TextView，绑定至id号为2130968576

。。。依此类推
```

对ButterKnife注解解析完成，生成`targetClassMap`中的对应关系之后，就可以动态的生成绑定相关的代码。

动态生成绑定代码由`bindingClass.brewJava()`实现。上面得到的`targetClassMap`中，每一个`bindingClass`对应一个Sample工程中的涉及ButterKnife注解的类。在`bindingClass.brewJava()`中，使用到了第三方类库[`JavaPoet`][javapoet]
还是以`SimpleActivity`为例，以下代码首先根据`bindingClass.className`生成相关的类：
```java
TypeSpec.Builder result = TypeSpec.classBuilder(className)
        .addModifiers(PUBLIC)
        .addTypeVariable(TypeVariableName.get("T", ClassName.bestGuess(targetClass)));
```

这里`className`=`SimpleActivity$$ViewBinder`，`targetClass`=`SimpleActivity`

`SimpleActivity`没有父类，于是通过以下代码给`SimpleActivity$$ViewBinder`设置父类`ViewBinder`

```java
  result.addSuperinterface(ParameterizedTypeName.get(VIEW_BINDER, TypeVariableName.get("T")));
```

于是我们到目前为止有了一个动态生成的类`SimpleActivity$$ViewBinder`

```java
public class SimpleActivity$$ViewBinder<T extends SimpleActivity> implements ViewBinder<T> {

}

```

之后以下语句为该类添加`bind`方法：

```java
  result.addMethod(createBindMethod());
```
打开`createBindMethod()`方法可以看到，首先构造了一个名为`bind`的方法，该方法有一个`@Override`注解，`pulbic`，三个参数分别为`final Finder finder, final T target, Object source`。

```java
MethodSpec.Builder result = MethodSpec.methodBuilder("bind")
        .addAnnotation(Override.class)
        .addModifiers(PUBLIC)
        .addParameter(FINDER, "finder", FINAL)
        .addParameter(TypeVariableName.get("T"), "target", FINAL)
        .addParameter(Object.class, "source");
```

对于`@Binde`注解的`View`类型的成员变量，处理这些元素的代码在这里

```java
for (ViewBindings bindings : viewIdMap.values()) {
        addViewBindings(result, bindings);
}
```

在`addViewBindings`方法中最终调用以下语句针对每一个被注解的成员变量生成具体的执行代码：

```java
result.addStatement("view = finder.findRequiredView(source, $L, $S)", bindings.getId(),
            asHumanDescription(requiredViewBindings));
```

```java
result.addStatement("target.$L = finder.castView(view, $L, $S)", fieldBinding.getName(),
            bindings.getId(), asHumanDescription(fieldBindings));
```

比如对于`SimpleActivity`中的成员变量`title`，其最终生成的执行代码为

```java
view = finder.findRequiredView(source, 2130968576, "field 'title'");
target.title = finder.castView(view, 2130968576, "field 'title'");
```

我们暂时可以先不用关心`findRequiredView`和`castView`的实现分别是什么。

等这些代码都构建完成后，调用`writeTo(filer);`将构建的类写入文件。最终动态生成的源码可在Sample工程构建完成后的`build\generated\source\apt\debug\com\example\butterknife`目录下找到。


绑定

绑定操作位于app运行时。通常在一个`Activity`的`onCreate`方法内调用`ButterKnife.bind(this);`执行。

在类`ButterKnife`有多个`bind`方法的重载，`Activity`中调用的重载版本是

```java
public static void bind(@NonNull Activity target) {
  bind(target, target, Finder.ACTIVITY);
}
```

注意这里的第三个参数`Finder.ACTIVITY`，它是枚举`Finder`下的一个枚举值，而`Finder`中声明了两个抽象方法

```java
protected abstract View findView(Object source, int id);

public abstract Context getContext(Object source);
```

又分别在包括`Finder.ACTIVITY`中的一系列枚举值当中做了实现。所以这里实际上可以认为Finder是一个抽象类，而`Finder.ACTIVITY`，`Finder.VIEW`，`Finder.DIALOG`是这个抽象类的实例，他们各自对以上两个抽象方法做了自己的实现。

后续的绑定流程中，`Finder.ACTIVITY`会以`Finder`类型的身份出现，当看到类似`finder.findView(source, id)`这样的语句时，我们就可以知道去哪里查看其内部实现。

在`bind(@NonNull Object target, @NonNull Object source, @NonNull Finder finder)`中，首先根据`target`的类型`targetClass`，即`SimpleActivity`找到其对应的`ViewBinder`，该操作位于`findViewBinderForClass(targetClass)`中。

在`findViewBinderForClass`方法中，针对每个`targetClass`如果是初次进入，会通过`Class.forName(String className)`方法动态加载其对应的`ViewBinder`类。这里`className`为`targetClass`的名字和`$$ViewBinder`拼接。以`SimpleActivity`为例，取到的类就是`com.example.butterknife.SimpleActivity$$ViewBinder`，即我们之前在`build\generated\source\apt\debug\com\example\butterknife`下动态生成的类。

如果`ViewBinder`类获取成功，获取其实例，以`targetClass`为key放入Map `BINDERS`中，下次再找`targetClass`对应的`ViewBinder`类实例时可直接在`BINDERS`中查找。返回`ViewBinder`类的实例。

取到了对应的`ViewBinder`实例之后，立即执行`viewBinder.bind(finder, target, source);`这里的`finder`是刚才的`Finder.ACTIVITY`，`target`和`source`都是调用`ButterKnife.bind`的`SimpleActivity`实例。

这里的`viewBinder.bind(finder, target, source);`执行的就是之前第二步中动态构造出来的方法，里面执行了一系列具体的`view`绑定操作，就是我们在第二步中暂时不用关心的那两行代码：


```java
view = finder.findRequiredView(source, 2130968576, "field 'title'");
target.title = finder.castView(view, 2130968576, "field 'title'");
```

现在我们需要了解这两个代码具体是怎么执行的绑定操作。

首先看`finder.findRequiredView(source, 2130968576, "field 'title'")`这个方法的调用，首先会`Finder.ACTIVITY`中的`findView(Object source, int id)`实现版本，可以看到该版本的`findView(Object source, int id)`将`source`强转为`Activity`类型后调用了它的 `findViewById(id)`方法，就是我们写到吐的那个方法。该方法返回了一个`View`类型变量。如果我们手写`findViewById(id)`的话，通常会对其返回值进行一次类型转换，转换为这个`View`的实际类型，比如最常见的`TextView`，`ListView`这种。而在这里，这种类型转换在`finder.castView(view, 2130968576, "field 'title'");`中进行（`findRequiredView`方法内虽然自带了一次`castView`调用，但是`findRequiredView`的返回值是`View`类型，所以这里的类型转换并没有起作用）。`castView`方法会根据它的模板类型`T`，即返回值的类型自行推断需要将`view`转换的目标类型。最后将返回值赋值给`target.title`即`SimpleActivity`的`title`成员变量，至此整个绑定操作完成。




以[https://github.com/knightingal/butterknife/tree/study-base][sample]中的Sample代码为例。





[javapoet]:https://github.com/square/javapoet
[sample]:https://github.com/knightingal/butterknife/tree/study-base
