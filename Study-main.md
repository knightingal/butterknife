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
  pos = 1111
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9732} "title"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9735}
 1 = {Symbol$VarSymbol@9715} "subtitle"
  pos = 1150
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9739} "subtitle"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9740}
 2 = {Symbol$VarSymbol@9716} "hello"
  pos = 1187
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9744} "hello"
  type = {Type$ClassType@9745} "android.widget.Button"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9746}
 3 = {Symbol$VarSymbol@9717} "listOfThings"
  pos = 1232
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9750} "listOfThings"
  type = {Type$ClassType@9751} "android.widget.ListView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9752}
 4 = {Symbol$VarSymbol@9718} "footer"
  pos = 1276
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9756} "footer"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9757}
 5 = {Symbol$VarSymbol@9719} "headerViews"
  pos = 1384
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9761} "headerViews"
  type = {Type$ClassType@9762} "java.util.List<android.view.View>"
  owner = {Symbol$ClassSymbol@9734} "com.example.butterknife.SimpleActivity"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9763}
 6 = {Symbol$VarSymbol@9720} "word"
  pos = 1486
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9767} "word"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9769}
 7 = {Symbol$VarSymbol@9721} "length"
  pos = 1524
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9773} "length"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9774}
 8 = {Symbol$VarSymbol@9722} "position"
  pos = 1566
  adr = -1
  data = null
  kind = 4
  flags_field = 0
  name = {UnsharedNameTable$NameImpl@9778} "position"
  type = {Type$ClassType@9733} "android.widget.TextView"
  owner = {Symbol$ClassSymbol@9768} "com.example.butterknife.SimpleAdapter.ViewHolder"
  completer = null
  erasure_field = null
  metadata = {SymbolMetadata@9779}
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
有两个类型中存在@Bind注解
第一个类型为com.example.butterknife.SimpleActivity
该类中有5个被@Bind注解的成员
第一个成员名字是title，类型是android.widget.TextView，绑定至id号为2130968576


依此类推
```

以[https://github.com/knightingal/butterknife/tree/study-base][sample]中的Sample代码为例。




[sample]:https://github.com/knightingal/butterknife/tree/study-base
