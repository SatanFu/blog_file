---
title: Activity生命周期监听
date: 2016-11-21 16:14:08
tags: Android
cover: http://ww3.sinaimg.cn/large/8a7693bbgw1f9zsvzqflyj20e90ifdhn.jpg
---

## 起因

最近在搞公司的用户行为跟踪系统的时候碰到要统计用户在Activity的停留时间，然后就想到了在生命周期里面添加时间记录。在每个Activity里面重写生命周期函数添加时间记录？太麻烦了吧。写个BaseActivity，在BaseActivity里面做时间记录？嗯，还行。还有么有更简便的方式。有啊，在Application里面调用registerActivityLifecycleCallbacks方法就可以监听所有Activity的生命周期，问题妥妥的搞定了。当时我在公司接触的项目都是4.0以上的了，所以也没考虑那么多。问题搞定了，是骡子是马来出来溜溜就知道了。然后领导就接入到公司一个比较旧的项目里面，然后过几天我就懵逼了，全都是registerActivityLifecycleCallbacks报的错误。当时表情是这样的

![黑人问号脸](http://ww2.sinaimg.cn/large/8a7693bbgw1f9zu6lj0xzj20b40ar74k.jpg)

后来发现旧项目最低支持2.3以上，所以只能用别的办法了，因此写下此文，记录一下解决办法。

## 解决之道

### 关键类 Instrumentation
Instrumentation这个类平时开发的时候可能接触的比较少，但如果弄过Android组件单元测试的话应该会接触过。这个类很强大，比如通过这个类我们可以在Activity创建之前做一些我们自己的事情，比如拦截掉启动的Activity，换成另外一个Activity，再比如获取Activity的生命周期。怎么获取的Activity生命周期？在Instrumentation类中有callActivityOnCreate、callActivityOnResume等一些生命周期的方法。既然有生命周期的这些方法，那我们只要Hook Instrumentation这个类，就可以实现Activity生命周期的监听。那怎么Hook呢，用反射啊。

### 实现方法
#### 0、首先先定义一个MyInstrumentation继承自Instrumentation，重写callActivityOnCreate、callActivityOnResume等Activity生命周期的方法
代码如下：

```java

public class MyInstrumentation extends Instrumentation {

	@Override
	public void callActivityOnCreate(Activity activity, Bundle icicle) {
		super.callActivityOnCreate(activity, icicle);
		ActivityLifeManager.getInstance().onActivityCreated(activity, icicle);
	}

	@Override
	public void callActivityOnPause(Activity activity) {
		super.callActivityOnPause(activity);
		ActivityLifeManager.getInstance().onActivityPaused(activity);
	}

	@Override
	public void callActivityOnDestroy(Activity activity) {
		super.callActivityOnDestroy(activity);
		ActivityLifeManager.getInstance().onActivityDestroyed(activity);
	}

	@Override
	public void callActivityOnResume(Activity activity) {
		super.callActivityOnResume(activity);
		ActivityLifeManager.getInstance().onActivityResumed(activity);
	}

	@Override
	public void callActivityOnStart(Activity activity) {
		super.callActivityOnStart(activity);
		ActivityLifeManager.getInstance().onActivityStarted(activity);
	}

	@Override
	public void callActivityOnStop(Activity activity) {
		super.callActivityOnStop(activity);
		ActivityLifeManager.getInstance().onActivityStoped(activity);
	}

}

```

注：ActivityLifeManager是自己定义的一个Activity生命周期管理类，方便管理4.0以上和4.0以下两种Activity生命周期监听方式

#### 1、然后将我们自己定义的MyInstrumentation利用反射将系统原本的Instrumentation替换掉

代码如下：

```java

/**
 * 替换系统默认的Instrumentation
 */
public static void replaceInstrumentation() {
	Class<?> activityThreadClass;
	try {
		// 加载activity thread的class
		activityThreadClass = Class.forName("android.app.ActivityThread");

		// 找到方法currentActivityThread
		Method method = activityThreadClass
				.getDeclaredMethod("currentActivityThread");
		// 由于这个方法是静态的，所以传入Null就行了
		Object currentActivityThread = method.invoke(null);

		// 把之前ActivityThread中的mInstrumentation替换成我们自己的
		Field field = activityThreadClass
				.getDeclaredField("mInstrumentation");
		field.setAccessible(true);
		MyInstrumentation mInstrumentation = new MyInstrumentation();
		field.set(currentActivityThread, mInstrumentation);
	} catch (Exception e) {
		e.printStackTrace();
	}
}

```

至此，4.0以下Activity生命周期监听难点已经解决，接下来就是整合4.0以上和4.0以下Activity生命周期监听。

#### 2、整合

ActivityLifeManager管理类，代码如下：

```java

public class ActivityLifeManager implements IActivityLifecycleCallbacks {

	private static ActivityLifeManager manager;
	private List<IActivityLifecycleCallbacks> lifeChanges = new ArrayList<IActivityLifecycleCallbacks>();

	public static synchronized ActivityLifeManager getInstance() {
		if (manager == null) {
			manager = new ActivityLifeManager();
		}
		return manager;
	}

	private ActivityLifeManager() {
	}

	public void addIActivityLifeChange(IActivityLifecycleCallbacks lis) {
		lifeChanges.add(lis);
	}

	@Override
	public void onActivityStarted(Activity activity) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityStarted(activity);
		}
	}

	@Override
	public void onActivityResumed(Activity activity) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityResumed(activity);
		}
	}

	@Override
	public void onActivityPaused(Activity activity) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityPaused(activity);
		}
	}

	@Override
	public void onActivityDestroyed(Activity activity) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityDestroyed(activity);
		}
	}

	@Override
	public void onActivityCreated(Activity activity,
			Bundle savedInstanceState) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityCreated(activity, savedInstanceState);
		}
	}

	@Override
	public void onActivityStoped(Activity activity) {
		for (IActivityLifecycleCallbacks lis : lifeChanges) {
			lis.onActivityStoped(activity);
		}

	}
}

```

IActivityLifecycleCallbacks生命周期回调接口，代码如下：

```java

public interface IActivityLifecycleCallbacks {

	public void onActivityCreated(Activity activity, Bundle savedInstanceState);

	public void onActivityStarted(Activity activity);

	public void onActivityResumed(Activity activity);

	public void onActivityPaused(Activity activity);

	public void onActivityStoped(Activity activity);

	public void onActivityDestroyed(Activity activity);

}

```

MyActivityLifecycleCallbacks 4.0以上Activity生命周期回调实现类

```java

public class MyActivityLifecycleCallbacks
		implements Application.ActivityLifecycleCallbacks {
	@Override
	public void onActivityCreated(Activity activity,
			Bundle savedInstanceState) {
		ActivityLifeManager.getInstance().onActivityCreated(activity,
				savedInstanceState);
	}

	@Override
	public void onActivityStarted(Activity activity) {

	}

	@Override
	public void onActivityResumed(Activity activity) {
		ActivityLifeManager.getInstance().onActivityResumed(activity);
	}

	@Override
	public void onActivityPaused(Activity activity) {
		ActivityLifeManager.getInstance().onActivityPaused(activity);
	}

	@Override
	public void onActivityStopped(Activity activity) {
		ActivityLifeManager.getInstance().onActivityStoped(activity);
	}

	@Override
	public void onActivitySaveInstanceState(Activity activity,
			Bundle outState) {

	}

	@Override
	public void onActivityDestroyed(Activity activity) {
		ActivityLifeManager.getInstance().onActivityDestroyed(activity);
	}
}

```

整合使用，代码如下：

```java

public static void init(Application app) {
	if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
    //4.0以上使用系统自带的Activity生命周期监听方式
		app.registerActivityLifecycleCallbacks(
				new MyActivityLifecycleCallbacks());
	} else {
    //4.0以下，替换Instrumentation，实现Activity生命周期监听
		replaceInstrumentation();
	}

  //最终两种方式生命周期统一回调
	ActivityLifeManager.getInstance()
			.addIActivityLifeChange(new IActivityLifecycleCallbacks() {

				@Override
				public void onActivityStarted(Activity activity) {

				}

				@Override
				public void onActivityResumed(Activity activity) {

				}

				@Override
				public void onActivityPaused(Activity activity) {

				}

				@Override
				public void onActivityDestroyed(Activity activity) {

				}

				@Override
				public void onActivityCreated(Activity activity,
						Bundle savedInstanceState) {

				}

				@Override
				public void onActivityStoped(Activity activity) {

				}
			});
}

```

### 源码地址 [ActivityLifecycle](https://github.com/SatanFu/ActivityLifecycle)

## 总结

![卧操，牛逼，啪啪啪](http://ww1.sinaimg.cn/large/8a7693bbgw1fa03x9a37lj205i057dfv.jpg)
