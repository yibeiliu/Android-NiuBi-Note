## EventBus使用简介

常用的使用方式如下：

```java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
      
        EventBus.getDefault().register(this);//注册
    }
  
    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);//解注册
    }

    @Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onSubscribe(EventBusEvent ev) {
				//粘性事件
    }

    @Subscribe(threadMode = ThreadMode.ASYNC)
    public void onSubscribe2(EventBusEvent ev) {
				//普通事件
    }

    public class EventBusEvent {
    }
}
```



## EventBus 源码解析

下面我将尝试从`EventBus.getDefault().register(this);`方法开始讲解。



```java
    static volatile EventBus defaultInstance;

		public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();
                }
            }
        }
        return instance;
    }

    public EventBus() {
        this(DEFAULT_BUILDER);
    }


    EventBus(EventBusBuilder builder) {
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
      //...
    }
```

EventBus 采用双重校验锁的单例写法，在构造方法中，传入了`DEFAULT_BUILDER`

```java
    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
```

这也是典型的建造者模式，因为builder的属性非常多，建造者模式正好完美契合这种需求。



往下看

```java
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

接下来在`register()`方法中，object类型的参数subscriber就是我们传的this，也就是例子中的 MainActivity。

找到他的class名，然后调用了非常关键的`subscriberMethodFinder.findSubscriberMethods(subscriberClass);` 这句看名字也可以猜到是根据class名找到有`@Subscribe`注解的订阅方法集合，我们具体看一下



首先，`subscriberMethodFinder` 是EventBus类初始化的时候new出来的，没什么好说的，我们看看具体的函数：

```java
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
    
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {//默认是false
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

METHOD_CACHE 是一个缓存的map，它记录了class和其对应的`SubscriberMethod`集合，`SubscriberMethod`没什么神秘的，我们只要知道它就是一个封装了Method，Class名的类而已。



整个函数的逻辑就是先找缓存，找到了就ok，没找到则调用`findUsingInfo(subscriberClass);`去找，如果没找到任何带有注解的函数，则抛出`EventBusException`异常

那么重点转移到`findUsingInfo(subscriberClass);`

```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

```java
    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];    
    private static final int POOL_SIZE = 4;

		private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```

这个函数可以说是EventBus的精髓所在了，首先我们要介绍下`FindState`这个类，这个类做了一些辅助操作，比如检验方法是否已经被添加过了等等。这里有一个新的东西`FIND_STATE_POOL`，它是 `FindState` 类型的数组，大小为4，我认为它是`FindState`类的复用池，如果在池子里找到存在的`FindState`对象，则拿过来直接用，如果不存在才new一个新的。

接着来到了 `getSubscriberInfo(findState);`

```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

在这里，决定`findState.subscriberInfo`是否为null取决于`subscriberInfoIndexes` 是否为 null，而`subscriberInfoIndexes`是EventBus的Builder创建的，我们使用默认Builder时该值一直为null(builder的详细使用可以参考文档)，所以这个函数的两个if都不会进入，直接返回null，那么进入了`findUsingReflectionInSingleClass(findState);`函数中

```java
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                if (ignoreGeneratedIndex) {
                    msg += ". Please consider using EventBus annotation processor to avoid reflection.";
                } else {
                    msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
                }
                throw new EventBusException(msg, error);
            }
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();//modifiers指的是权限修饰符
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
              //第一个条件是modifiers必须是public
              //MODIFIERS_IGNORE = Modifier.ABSTRACT | Modifier.STATIC | BRIDGE | SYNTHETIC;
							//第二个条件是modifiers必须不能是abstract和static以及bridge和synthetic，bridge和synthetic是编译器自动生成的，EventBus当然要过滤掉这种方法
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {//只有一个参数
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {//如果方法声明了@Subscribe却不是一个参数，则抛出异常
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {//这里也是一样，如果方法声明了@Subscribe但权限修饰符不对，则抛出异常
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

熟悉的反射，这里EventBus就会开始找MainActivity所声明的method了，具体的解析Method我在注释标明了，大家可以看下注释。

在一切都满足要求之后，执行了这一句

```
findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
        subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
```

也就是将通过反射找到的带有`@Subscribe`注解的方法封装成`SubscriberMethod`对象放到`FindState`的集合中去保存，到这里第一遍while循环就差最后一步了，还记得while循环的代码吗？我再贴一遍：

```java
    private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
            findState.subscriberInfo = getSubscriberInfo(findState);
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                findUsingReflectionInSingleClass(findState);//我们讲到这里啦！！！
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

在把当前类的method都保存好了之后，执行了`findState.moveToSuperclass()；`

```java
    void moveToSuperclass() {
        if (skipSuperClasses) {
            clazz = null;
        } else {
            clazz = clazz.getSuperclass();
            String clazzName = clazz.getName();
            // Skip system classes, this degrades performance.
            // Also we might avoid some ClassNotFoundException (see FAQ for background).
            if (clazzName.startsWith("java.") || clazzName.startsWith("javax.") ||
                    clazzName.startsWith("android.") || clazzName.startsWith("androidx.")) {
                clazz = null;
            }
        }
    }
```

这就是找当前class的父类，如果找到了就把clazz指向父类，从新开始新的while循环去寻找父类的符合条件的函数，如果没有符合要求的父类，则直接结束while循环进入`getMethodsAndRelease(findState);`

```java
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

这段代码应该熟悉了吧，recycle函数就是重新把`findState`的属性全部初始化，之后`findState`就被填入复用池中等待下一次被重用。这里return的是MainActivity和其父类所有的`@Subscribe`方法的集合。

还记得return到哪个函数了吗？哈哈

```java
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);//return在这里！！！！！
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

接下来的环节，就是遍历这个`SubscriberMethod`集合

```java
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }

```

这个方法有点长，不过功能很简单，就是把每个方法绑定起来，这里是一个观察者模式，读者可以好好领会。

下面详细说下这个方法，这里需要极其注意的点就是：

```java
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    private final Map<Object, List<Class<?>>> typesBySubscriber;
```

这两个Map是EventBus观察者的核心map，比方说前面的例子，MainActivity中订阅了EventBusEvent这个事件，那么

> subscriptionsByEventType 保存了EventBusEvent和订阅的Method之间的关系

> typesBySubscriber 保存了订阅类即MainActivity和参数类即EventBusEvent之间的关系

理解了这两个的作用，我们就看下这个函数的作用。首先判断`subscriptionsByEventType`中有没有`eventType`对应的Method，`eventType`也就是我们所说的订阅类(EventBusEvent)，如果有，判断新来的订阅事件是否已经存在，如果存在就抛出异常（由于`SubscriberMethod`和`Subscription`都重写了equals方法，所以这里只有当方法的声明类，方法的名字以及订阅类名字完全一致才会抛出异常，但是正常情况下，编译器是不允许一个类里出现同名的函数的）

接下来，

```java
        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
```

则是根据priority的优先级排序，优先级高的放在list的前面，如果最低，则插在list尾部，到此为止，subscriptionsByEventType这个map就关联完毕。下面开始关联typesBySubscriber

```java
List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);//订阅类和method之间建立map关联
        }
        subscribedEvents.add(eventType);
```

那好，接下来我们看到下面的代码和sticky有关，那么也就是粘性事件了

```java
        if (subscriberMethod.sticky) {
            if (eventInheritance) {
                // Existing sticky events of all subclasses of eventType have to be considered.
                // Note: Iterating over all events may be inefficient with lots of sticky events,
                // thus data structure should be changed to allow a more efficient lookup
                // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
```

如果`eventInheritance`为false(默认就是false)，则从map中找到对应的对象，去执行`checkPostStickyEventToSubscription`，如果`eventInheritance`为true，则会去map里找该订阅类的所有

子类是否被注册，找到则执行`checkPostStickyEventToSubscription`



那么显而易见，`checkPostStickyEventToSubscription`会调用`postToSubscription`

```java
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
                if (isMainThread) {
                    backgroundPoster.enqueue(subscription, event);
                } else {
                    invokeSubscriber(subscription, event);
                }
                break;
            case ASYNC:
                asyncPoster.enqueue(subscription, event);
                break;
            default:
                throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
        }
    }

		//通过反射执行订阅的方法
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```

是不是豁然开朗？这就是根据`@Subscribe`注解的`ThreadMode`去决定到底在哪个线程执行回调

- POSTING

  在哪个线程发的消息，就在哪个线程执行回调

- MAIN

  在主线程执行

- MAIN_ORDERED

  在主线程执行，它和MAIN的区别在于，MAIN_ORDERED是通过handler发消息去执行的，也就是不存在阻塞，试想一下，如果EventBus的订阅方法里有耗时或者阻塞的方法，那么通过Handler则可以先发出消息然后返回，真正的执行则可以等待阻塞完毕后再执行，而MAIN是直接在主线程执行反射，不会返回导致程序卡死

- BACKGROUND

  如果当前是主线程，则通过线程池开启线程执行，如果当前不是主线程，立刻在当前线程执行

- ASYNC

  一定会开启线程去执行

#### register方法解析

值得一提的是，EventBus的线城池采用的是`newCachedThreadPool`

到此为止，register的功能解析就完成了，总结一下，register主要做了以下几件事，

- 找到当前类的所有带`@Subscribe`注解且符合要求的方法，存到`subscriptionsByEventType`和`typesBySubscriber`这两个map中
- 如果带注解的方法中存在sticky，那么直接从`stickyEvents`中找出存好的对象进行反射调用，完成sticky的功能



#### post方法解析

```java
    private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    
		public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

`currentPostingThreadState`是一个`ThreadLocal`对象，而`PostingThreadState`中则有一个队列`eventQueue`，那么这就说明在每一个线程中都有一个`PostingThreadState`对象，而`PostingThreadState`对象一一对应了`eventQueue`，也就是每个线程都有一个`eventQueue`，我们把event放入`eventQueue`中，然后循环这个queue，每循环一次取出一个事件执行`postSingleEvent` (*这里我不太理解为什么要设置一个队列，理解的小伙伴评论里告诉我下，感激不尽~*)

```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```
由于`eventInheritance`前面我们说过是跟订阅类继承关系有关，默认是false，所以我们直接跳到`postSingleEventForEventType`方法
```java
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted;
                try {
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```

出现了！！是`subscriptionsByEventType`！！还记得吗，这事订阅类和method之间的map，我们找到订阅类的所有method信息，然后遍历去执行`postToSubscription`，这个方法前面讲sticky的时候遇到过，其实反射执行的代码只有这么一处，无论sticky还是普通的event，最后反射的部分都是一样的。

#### postSticky解析

```java
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```

看起来很简单，调用了刚才说过的post()方法，只不过在这之前多了一步，stickyEvents这个map之前也说到过，当时是在register时会遍历这个map找到匹配的event去执行，那么这么map便是此时赋值的，这也符合sticky的行为：在普通event的基础上，增加了订阅后立即调用的功能。

#### unregister解析

unregister就非常简单了，我们猜也猜出来了，无非是把一堆之前提到的map里包含当前类的value全部清空

```java
    public synchronized void unregister(Object subscriber) {
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
                unsubscribeByEventType(subscriber, eventType);
            }
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }

    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }
```

看到了吧，和我们猜的一样，`subscriptionsByEventType`和`typesBySubscriber`这两个map中相关联的value全部清除了。

#### 总结

到这里EventBus的基本原理就解析完了，我们了解了EventBus的基本功能是怎么实现的，普通event和粘性event的实现方式，还知道了可以支持像订阅类继承这种功能，其实在Builder中还有很多配置，我们可以参考文档去学习，至于其原理，我相信主线流程和思想都懂了，那些东西也不是什么难事。

当然我们看开源库，不仅要了解它的原理，还要尽量从中学到一些技巧。EventBus中就用到了单例，建造者模式，观察者模式，还有对象复用池的思想，反射和注解的使用，这些相信看完了文章的你应该都有所了解了吧。