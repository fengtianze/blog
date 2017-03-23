# Android 官方 MVVM 框架：DataBinding 实例讲解（一）

## 引子

前一段开始学习并在项目中运用 MVVM 时，首先学习的便是 DataBinding 这个 Google 爸爸官方出品的数据绑定框架。我在网上查资料时发现，DataBinding 相关的资料虽然不少，但大多是在 DataBinding 推出不久时写的，有很多之后新加入的功能都没有提到，内容略显陈旧，而 Android 开发者官网提供的[官方文档](https://developer.android.com/topic/libraries/data-binding/index.html)也不并全面，有些很重要的知识点没有提到。所以我想结合官方文档和自己的实际使用经验，向大家介绍一下 DataBinding 的使用方法。在这个系列完成后，我想再聊聊 RxJava、Retrofit、DataBinding 等相结合打造 MVVM 架构的方法。

如果你在阅读时发现错误，或者有什么建议，欢迎反馈给我。

## DataBinding 框架简介

DataBinding 是 Google 在2015年7月推出的一个支持库，最低兼容至 Android 2.1(API level 7+)，需要 Gradle 版本高于 1.5.0-alpha1。

Google 一直在增强 DataBinding 的功能以及 Android Studio 对其的支持度，目前 Studio 已经支持了大部分的代码补全和错误提示，但仍有部分正确语法会被报错。Anyway，我强烈建议你保持 Android Studio 和 Gradle 为最新稳定版。

## 配置环境

首先确保你已下载安装最新版 `Support Repository`，接下来在 `module` 层级的 `build.gradle` 文件中添加以下配置代码：

```
android {
    ...
    dataBinding {
        enabled = true
    }
}
```

## 绑定数据

首先创建一个标准的 `JavaBean`：

```
//  Person.class

public class Person {

    private String name;

    //  省略的 getter 和 setter 方法
}
```

然后需要编写布局文件。将根元素改为 `layout`，在其中添加用于包裹数据的 `data` 元素和正常的布局代码。
一个编写好的布局类似如下代码：

```
//  activity_main.xml

<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <variable
            name="person"
            type="com.spacerover.databindingdemo.data.Person"/>
    </data>

    <LinearLayout
        android:id="@+id/activity_main"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/nameTv"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{person.name}"
            android:textAlignment="center"/>

        <EditText
            android:id="@+id/nameEt"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{person.name}"/>
    </LinearLayout>
</layout>
```

代码中的 `variable` 用于定义数据绑定所需要的变量，其中 `name` 属性定义变量名，`type` 定义变量类型。
示例中将 `person` 声明为位于 `com.spacerover.databindingdemo.data` 包下的 `Person`，也就是我们刚刚写的 `JavaBean`。
使用 `@{}` 语法可以将变量绑定到控件的属性。

接下来需要编写布局文件相对应的 `Activity`。像这样重写 `onCreate(...)` 方法：

```
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    //  替换 setContentView(R.layout.activity_main);
    ActivityMainBinding binding = DataBindingUtil.setContentView(this,R.layout.activity_main);
    final Person person = new Person();
    person.setName("张三");
    binding.setPerson(person);
    new Handler().postDelayed(new Runnable() {
        @Override
        public void run() {
            person.setName("李四");
        }
    }, 3000);
}
```

其中，`ActivityMainBinding` 是根据之前编写的布局文件 `activity_main.xml` 自动编译生成的类。它接管了布局文件中声明的所有绑定，我们可以通过 `binding.setPerson(...)` 或 `binding.getPerson()` 方法设置、访问绑定的变量。

此外，`binding` 还带来了另一个重要福利，这一知识点本应放在之后介绍，但我忍不住现在就告诉你：我们现在终于可以与 `findViewById(...)` 和 `ButterKnife` 说再见了！`binding` 会自动通过 `id` 创建对控件的引用。在本例中我们可以通过 `binding.nameEt` 直接访问 `id` 为 `nameEt` 的 `EditText`！虽然它不能直接访问到通过 `include` 添加进布局的控件，但我们可以给被 `include` 的布局添加 `layout` 元素从而间接访问其中的控件。

回到正题，除了示例中的方法，我们还可以通过以下方法在不同组件中添加绑定：

```
//  Activity
ActivityMainBinding binding = MainActivityBinding.inflate(getLayoutInflater());

//  Fragment
FragmentPageBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.fragment_page, viewGroup, false);
//  or
FragmentPageBinding binding = FragmentPageBinding.inflate(layoutInflater, R.layout.fragment_page, viewGroup, false);

//  ListView or RecyclerView
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
//  or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
```

至此，我们已经可以运行程序查看效果了。运行之后可以看到界面上成功显示出「张三」，但3秒后并没有显示「李四」。这是因为单纯的使用 `JavaBean` 绑定数据时，`DataBinding` 无法自动刷新界面。此时我们需要对 `JavaBean` 进行一些改造，首先继承 `BaseObservable` 类，然后在 `getter` 方法之前添加 `Bindable` 注解，在 `setter` 方法之内添加 `notifyPropertyChanged(...)` 方法，就像这样：

```
//  Person.class

public class Person extends BaseObservable{

    private String name;

    @Bindable
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
        notifyPropertyChanged(BR.name);
    }
}
```

再次运行代码，界面应该可以正常显示「李四」了。但使用这种方法比较麻烦，代码量偏高，而且在实际使用时编译环境可能经常报错，所以我推荐使用 `DataBinding` 提供的另一种方式：`ObserableField`。

除了 `BaseObservable`，`Databinding` 还向我们提供了 `ObserableField` 用于快速创建绑定。`ObserableField` 通过泛型的方式实现对对象的支持，对于其它数据类型则实现了：`ObserableBoolean`，`ObserableByte`，`ObserableChar`，`ObserableShort`，`ObserableInt`，`ObserableLong`，`ObserableFloat`，`ObserableDouble` 和 `ObserableParcelable`。我们可以这样使用：

```
//  Pserson.class

public class Person {

    public final ObservableField<String> name = new ObservableField<>();
    public final ObservableInt age = new ObservableInt();
}
```

当访问变量时只需：

```
person.name.set("张三");
int age = person.age.get();
```

看起来不管用什么方法创建绑定都很简单，但你可能已经发现一个问题：从 `EditText` 修改名字时 `TextView` 无法同步刷新，而 `pserson.name` 的值也没有变化！修复这个问题实现数据的双向绑定非常简单。让我们回到布局文件，然后将给 `EditText` 绑定数据时使用的 `@{}` 语法改为 `@={}`，然后，就没然后了！数据的双向绑定就此完成！

## 尾巴

我计划在下一篇文章里继续介绍 `DataBinding` 的高级用法，如命令绑定等。