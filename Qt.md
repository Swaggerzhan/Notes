# Qt exp



### UI文件



xxx.ui文件中设计的界面最终都将被转换成一个头文件，程序想使用这个设计好的`ui`都需要通过这个来访问，头文件的命名规则一般为`ui_xxx.h`的，内部的类名则为`Ui_Xxxx`(首字母大写)，这个所谓的文件将在qmake编译时才会动态生成的。

如果我们选择一个简单的`Widget`的项目，则会生成如下的代码

```c++
#include "widget.h"
#include "ui_widget.h"

Widget::Widget(QWidget *parent)
    : QWidget(parent)
    , ui(new Ui::Widget)
{
    ui->setupUi(this);
}

Widget::~Widget()
{
    delete ui;
}
```

其中`ui_widget.h`就是连接我们设计的ui的关键，而构造列表中可以明显看到使用了`QWidget`来初始化一个父类，以及初始化一个`ui`，这里看似很奇怪，因为`ui(new Ui::Widget)`不是递归吗？其实在`ui_widget.h`中是有一行代码如下的:

```c++
namespace Ui {
    class Widget: public Ui_Widget {};
} // namespace Ui
```

也就是说，我们将`Ui_Widget`看似的定义成了另外的类名`Widget`，并且是处于`Ui`这个命名空间下的，而我们再看`widget.h`头文件即可明白

```c++
#include <QWidget>

QT_BEGIN_NAMESPACE
namespace Ui { class Widget; } // 声明
QT_END_NAMESPACE

class Widget : public QWidget
{
    Q_OBJECT

public:
    Widget(QWidget *parent = nullptr);
    ~Widget();

private:
    Ui::Widget *ui; // 自己
};
```

在代码中的namespace其实只是一个声明而已。

这样代码逻辑就成了，我们设计一个ui，ui被qmake操作后生成了一个文件，名为`ui_widget.h`，其中定义了一个类`Ui_Widget`，这个类就是ui设计的代码实现，并且在最后， __我们通过定义另外一个处于`Ui`这个命名空间中的一个类`Widget`继承于`Ui_Widget`__ 。

然后在最外层的`Widget`中使用在`Ui`这个命名空间中的`Widget`的`setupUi`将本身注册进去，在`setupUi`这个函数中，就是设计的ui的代码实现(ui中的某些窗口的位置，大小，名字，信号和槽等等)。

之后我们需要写的代码都可以直接写到外层的这个`Widget`类中，并且通过`ui`指针来去获取在设计中的一些组件等等。



### 信号和槽

这种关系有点类似Linux中的signal和signal handler的关系，槽即位函数嘛，具体定义方式为

```c++
class Widget{
  Q_OBJECT// 宏
public slots: // 权限可自由定义
	void slot_func();
};
```

其中`Q_OBJECT`宏是信号和槽的基础，如果一个类中需要使用到信号和槽，则需要`Q_OJBECT`宏，这将由MOC进行预处理。



##### 1. 信号的发送者

当一个槽函数被调用时，如果我们想知道信号的发送者，则可以通过`sender()`来获得，并且通过`qobject_cast`来对其进行转换，比如

```c++
QSpinBox* spinBox = qobject_cast<QSpinBox*>(sender()); // 获取信号发射者
```

##### 2. 自定义信号

使用`sigals`关键字可以自定义信号，并且可以通过`emit`来手动控制信号的发送，其中，信号函数不可以有返回值，但是可以有参数。比如

```c++
class Test{
	Q_OBJECT
public:
signals:
  void mySignal(int value);
private:
  void sendMySignal(){
    emit mySignal(10); // 手动发射信号出去
  }
};
```



### 一些组件



##### 1. QLineEdit

一个编辑文本的组件，具体的使用函数如下

```c++
lineEdit->setPalette(QPalette); // 通过Palette设定颜色
lineEdit->text();								// 获取一个QString的对象，就是文本中的内容
```

其中， __只有一个信号，那就是文本改变信号`QLineEdit::textChange` __，当文本发生改变时会被触发。

##### 2. QLabel

一段文本，改变颜色只可以发生在初始化的时候，一旦显示了，就不能再改变了。

##### 3. RadioButton

这种是一个互斥的选项，例如选中了A，则B被自动取消。具体使用的一些函数

```c++
radioButton->isChecked(); // 标明是否被选中，被选中则为true
```

信号

```c++
clicked(); // 被点击后的信号
```



##### 5. QSpinBox

这是一个数值的组件，有可以手动输入，也可以通过上下箭头来增减数量，一些函数

```c++
spinBox->setRange(int, int); // 设置范围
spinBox->setSingleStep(int); // 设置一步的长度
spinBox->setValue(int);			 // 默认值
```

信号有2种，都是关于改变时触发，分别是:

```c++
QSpinBox::valueChanged(int i);
QSpinBox::valueChanged(const QString &text);
```

* QSpinBox的信号和槽关联方式

由于`valueChanged`是一个同名的函数，并且有不同参数，在Qt中，这种情况就不能使用函数指针来进行connect了，但可以使用如下的方式进行关联。

```c++
// 其中，int将会是这个组件改变后的新值，将被传递到onValueChanged中的int。
connect(ui->spinBox, SIGNAL(valueChange(int)), this, SLOT(onValueChanged(int)));
```

