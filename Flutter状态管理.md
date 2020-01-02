​                                                             Flutter 状态管理



一：State,StatelessWidget,StatefulWidget

   （1）Flutter中万物皆Widget，但是用户界面显示无非就2中，静态的界面，动态的界面（数据状态驱动）

  A.StatelessWidget

​    静态的widget,一次build之后，不会再进行任何的修改。从源码中我们可以直观的看出

```
/// A widget that does not require mutable state.
abstract class StatelessWidget extends Widget {
  /// Initializes [key] for subclasses.
  const StatelessWidget({ Key key }) : super(key: key);

  /// Creates a [StatelessElement] to manage this widget's location in the tree.
  ///
  /// It is uncommon for subclasses to override this method.
  @override
  StatelessElement createElement() => StatelessElement(this);
  ///
  ///  * [StatelessWidget], which contains the discussion on performance considerations.
  @protected
  Widget build(BuildContext context);
}
```

因此它不需要维护State。

B.StatefulWidget

用途：再整个widget生命周期内本身持有的数据会发生变化而导致用户界面会发生响应的变化，因此被称为Stateful Widget（比如:网络请求，用户行为，或者业务逻辑等等）。

```
abstract class StatefulWidget extends Widget {
  /// Initializes [key] for subclasses.
  const StatefulWidget({ Key key }) : super(key: key);

  /// Creates a [StatefulElement] to manage this widget's location in the tree.
  ///
  /// It is uncommon for subclasses to override this method.
  @override
  StatefulElement createElement() => StatefulElement(this);

  /// Creates the mutable state for this widget at a given location in the tree.
  ///
  /// Subclasses should override this method to return a newly created
  /// instance of their associated [State] subclass:
  ///
  /// ```dart
  /// @override
  /// _MyState createState() => _MyState();
  /// ```
  ///
  /// The framework can call this method multiple times over the lifetime of
  /// a [StatefulWidget]. For example, if the widget is inserted into the tree
  /// in multiple locations, the framework will create a separate [State] object
  /// for each location. Similarly, if the widget is removed from the tree and
  /// later inserted into the tree again, the framework will call [createState]
  /// again to create a fresh [State] object, simplifying the lifecycle of
  /// [State] objects.
  @protected
  State createState();
}
```

从源码中我们可以看出，子类必须实现父类createState()方法

C.State

定义了StatefulWidget实例的行为，从而直接影响Widget的行为和布局

![未命名文件 (1)](/Users/wangyazhou/Downloads/未命名文件 (1).png)

基于State的任何改变都会导致当前的Widget进行重建操作，因此如何正确的使用这些状态，在必要的时候去重建布局就显得尤为重要了。

二：为什么要状态管理

Flutter作为一个声明式框架，我们也许只需要将数据映射成视图就可以了，如果我们的APP比较简单的话，比如：

![未命名文件 (2)](/Users/wangyazhou/Downloads/未命名文件 (2).png)

但是如果我们的APP足够复杂，比如：

![app_ful_state](/Users/wangyazhou/Downloads/app_ful_state.png)

随着我们APP功能越来越复杂，如果我们一个状态的改变（播放音乐的状态），可能我们只需要在DeviceA UI 和Songs List这里共享，但是由于我们的状态是层层传递的，因此在同级别的UI父节点下的子节点都会执行rebuild操作，这显然大多数UI进行rebuild操作都是多余的，进行大量的重绘操作，导致体验和性能变差。

另外这种方式会导致父组件的State会越来越臃肿，显然有些数据并不适合把他放在State里面作为共享数据，而且随着我们业务的多样化，真个父组件的State将变得越来越难以维护。因此我们需要进一步的手段来进行状态管理。

### 二：区分全局状态与临时状态

![未命名文件 (5)](/Users/wangyazhou/Downloads/未命名文件 (5).png)

 从上图我们可以看出：

1.临时状态：资源的收藏。

2.全局状态：资源的播放状态

从图中我们可以看出，对于资源的播放状态的管理，我们应该升级到最顶级的父组件中去管理，二对于资源的收藏，我们只需要在Resource Page去管理就可以了。

因此如何正确的管理状态时机去重建UI就显得尤为重要了 。

#### 没有明确的界限去区分

我们所谓的的临时状态与全局状态总是随着我们自己业务变换而变换的。比如可能后面业务的发展我们的播放状态不需要再其他组件中再使用，那么我们就需要把响应的状态做降级处理以适应业务的需求。

三：Flutter提供的状态管理

（一）特殊的数据共享

InheritedWidget是一种特殊的功能性组件，它提供了一种将状态从上向下传递的方式。

![未命名文件 (4)](/Users/wangyazhou/Downloads/未命名文件 (4).png)

项目中典型的应用：APP的Theme管理。

在一个MaterialApp中，我们可以在build任意widget时使用`ThemeData data = Theme.of(context)`来获取当前主题数据，同时也会使当前Widget依赖了Theme（准确地说是`_InheritedTheme`这一组件），当主题变化时，所有依赖它的Widget都会被更新。这就解决了直接传递State导致多余的组件更新的问题。因此如果只是共享数据采用自上而下的方式，子控件不需要任何修改的情形下还是不错的。

缺点：

​      由于InheritedWidget只是提供了一套自上而下的传递方式，如果子控件需要修改状态传递到父控件的话，显然就不能满足要求了。

(二)：Provider(Flutter官方推荐的状态管理组件)

我们来看一个简单的demo

```
class DeviceModeManager extends ChangeNotifier {
  List<Device> devices = [];

  void add(Device device) {
    devices.add(device);
    notifyListeners();
  }
}

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  final deviceManager = DeviceModeManager();

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider.value(
        value: deviceManager,
        child: MaterialApp(
          title: 'Flutter Demo',
          theme: ThemeData(
            primarySwatch: Colors.blue,
          ),
          home: MyHomePage(title: 'Flutter Demo Home Page'),
        )
    );
  }
}

class DeviceListPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final _counter = Provider.of<DeviceModeManager>(context);
    return Scaffold(
      appBar: AppBar(
        title: Text("Device list"),
      ),
      body: ListView.builder(itemBuilder: (BuildContext context, int index) {
        return Text(_counter.devices[index].name);
      },
        itemCount: _counter.devices.length,
      ),
      floatingActionButton: RaisedButton(
        onPressed: (){
          _counter.add(Device("new device"));
        },
        child: Text(
          "添加设备"
        ),
      ),
    );
  }
}
```

从代码中可以看到，在代码开始我们想Widget树种注入了ChangeNotifierProvider，我们可以从 Provider.of<DeviceModeManager>(context) 来拿到我们需要共享的数据，同时可以修改数据，然后其他订阅了此消息的widget会进行重建操作。

其中可以很明显的看出InheritedWidget方便太多了，InheritedWidget如果我们在子Widget里面想要去更新数据，那么只能通过Notification来自己实现。

代码中我们用到了ChangeNotifierProvider，这类似于ScopedModel（一个类似于Provider的插件），但是它唯一的缺陷就是我们是使用时必须继承它提供的Model才能使用。而系统Provider就响应的简单多了。

当然我们的APP不可能只管理一种状态，因此Flutter提供了MultiProvider来让我们更多的管理APP的状态

```
class MultiProvider extends StatelessWidget
    implements SingleChildCloneableWidget {
  /// Build a tree of providers from a list of [SingleChildCloneableWidget].
  const MultiProvider({
    Key key,
    @required this.providers,
    this.child,
  })  : assert(providers != null),
        super(key: key);

  /// The list of providers that will be transformed into a tree from top to
  /// bottom.
  ///
  /// Example: with [A, B, C] and [child], the resulting widget tree looks like:
  ///   A
  ///   |
  ///   B
  ///   |
  ///   C
  ///   |
  /// child
  final List<SingleChildCloneableWidget> providers;

  /// The child of the last provider in [providers].
  ///
  /// If [providers] is empty, [MultiProvider] just returns [child].
  final Widget child;

  @override
  Widget build(BuildContext context) {
    var tree = child;
    for (final provider in providers.reversed) {
      tree = provider.cloneWithChild(tree);
    }
    return tree;
  }

  @override
  MultiProvider cloneWithChild(Widget child) {
    return MultiProvider(
      key: key,
      providers: providers,
      child: child,
    );
  }
}

```

同时Flutter还提供了另外的集中形式的进行数据管理：

1. Provider
   - 单纯共享数据给子组件，但是数据更新时不会通知子组件。
2. [ListenableProvider](https://pub.dartlang.org/documentation/provider/latest/provider/ListenableProvider-class.html)
   - 比起ChangeNotifierProvider区别主要是不会自动调用`ChangeNotifier.dispose`释放资源。一般不用。
3. ValueListenableProvider
   - 可以认为是ChangeNotifierProvider的特例，只监听一个数据的时候，使用ValueListenableProvider在修改数据时可以不用调用`notifyListeners()`。
4. [StreamProvider](https://pub.dartlang.org/documentation/provider/latest/provider/StreamProvider-class.html)
   - 用于监听一个Stream
5. [FutureProvider](https://pub.dartlang.org/documentation/provider/latest/provider/FutureProvider-class.html)
   - 提供了一个 Future 给其子孙节点，并在 Future 完成时，通知依赖的子孙节点进行刷新

四：如何进行状态管理（现在市面上的案例fish_redux,redux）

1.redux

Redux是一种**单向数据流**架构，可以轻松开发，维护和测试应用程序。

![未命名文件 (6)](/Users/wangyazhou/Downloads/未命名文件 (6).png)

a.Reducer

```
typedef State Reducer<State>(State state, dynamic action);
```

用于根据 Action 来产生新的状态

b.Action

用于定义数据变化的行为

c.Store

```
Store(
  this.reducer, {
  State initialState,
  List<Middleware<State>> middleware = const [],
  bool syncStream = false,

  /// If set to true, the Store will not emit onChange events if the new State
  /// that is returned from your [reducer] in response to an Action is equal
  /// to the previous state.
  ///
  /// Under the hood, it will use the `==` method from your State class to
  /// determine whether or not the two States are equal.
  bool distinct = false,
}
```

用于存储和管理 state

![未命名文件 (7)](/Users/wangyazhou/Downloads/未命名文件 (7).png)

（1）首先APP所有的状态都是通过Store管理的，因此在程序开始，我们需要将Store放在widget树顶层

```
void main() {
  // Create your store as a final variable in the main function or inside a
  // State object. This works better with Hot Reload than creating it directly
  // in the `build` function.
  final store = Store<int>(counterReducer, initialState: 0);

  runApp(FlutterReduxApp(
    title: 'Flutter Redux Demo',
    store: store,
  ));
}
```

（2）**View**拿到Store储存的状态(State)并把它映射成视图。View还会与用户进行交互，用户点击按钮滑动屏幕等等，这时会因为交互需要数据发生改变。

（3）Redux让我们不能让View直接操作数据，而是通过发起一个**action**来告诉**Reducer**，状态得改变啦。

（4）这时候**Reducer**接收到了这个**action**，他就回去遍历action表，然后找到那个匹配的action，根据action**生成新的状态**并把新的状态放到Store中。

（5）Store丢弃了老的状态对象，储存了新的状态对象后，就通知所有使用到了这个状态的View更新（类似setState）。这样我们就能够同步不同view中的状态了。

2.fish_redux

fish_redux的学习成本其实是这几个状态管理里面最高的，但是带来的收益和效果也是最明显，fish_redux是基于redux封装，不仅仅能够满足状态管理，更是集成了配置式的组装项目，Page组装，Component实现，非常干净，易于维护，易于协作没，将集中、分治、复用、隔离做的更进一步，缺点就是代码量的急剧增大。

五：我们的项目是如何进行状态管理