---
title: android-Activity与Fragment
date: 2016-08-18 10:54:04
categories: android
tags: ["android", "Activity", "Fragment", "ActivityThread"]
---

# Activity的启动模式（android:launchMode）
1. standard 标准模式
	>系统默认模式，每次启动ACtivity，不管该Activity的实例是否存在，都会创建一个新的实例。**该Activity与启动它的Activity属于同一个任务栈**。
	>
	>注意：applicationContext不可启动standard模式的Activity。因为非Activity的context类型没有任务栈，那么如果要启动该模式的Activity，那该Activity无法进入任务栈，那么该Activity就不属于任何任务栈。
	
2. singleTop
	>栈顶复用模式。如果将要启动的Activity**处于任务栈的顶部**，那么该Activity会直接复用栈顶的实例。因为该Activity的实例已经存在，那么便不会执行onCreate和onStart，但会调用**onNewIntent**方法。在**onNewIntent**中可以取出这次请求的信息。但是如果该Activity不在任务栈的顶部，则会新建该Activity的实例。

3. singleTask
	>栈内复用模式。如果即将启动的Activity在某个任务栈中存在，则直接复用该Activity实例。再次过程中，只会执行**onNewIntnent**。具体而言，当存在该Activity所需要的任务栈，那么在该任务栈中查找该Activity的实例，如果查找到则直接复用，否则新建该Activity的实例并入栈。如果不存在该Activity的任务栈，则新建一个任务栈，然后新建Activity实例，并入栈。
	
	**注意：**当栈内复用时，那么将会把栈内该Activity之上的所有实例出栈。 
4. singleInstance
	>栈内单例模式。如果某个Activity为该模式，那么该Activity只能单独的存在于某一个任务栈中，其他特征与singleTask相同。
	
**注意：** 上述任务栈是由taskAffinity属性来判定的，默认情况下所有的Activity的taskAffinity属性都是应用包名。如果想指定某个Activity所在的任务栈，修改该属性即可。该属性仅仅在于singleTask或allowTaskReparenting配合使用时才有意义。

附：FLAF vs android:launchMode

FLAG_ACTIVITY_NEW_TASK:singleTask

FLAG_ACTIVITY_CLEAR_TOP: 如果该Activity在当前任务栈中存在实体，那么销毁栈中在其之上的Activity实例，然后创建一个新的实例添加到栈顶。

FLAG_ACTIVITY_SINGLE_TOP: singleTop

总结：使用FLAG则不支持singleInstance，使用android：launchMode则不支持CLEAR_TOP

# Task字段说明
1. Android:allowTaskReparenting
	> 该属性用来标记当Activity退居后台之后，是否能从启动它的Activity所在的任务栈移动到与它有相同taskAffinity的任务栈。eg：在应用中页面A启动了浏览器，那么此时打开的浏览器页面B与A是属于同一个任务栈的，当B退居后台之后，B就会移动到浏览器所在的任务栈中，并且处于该栈的顶部，所以当再次打开浏览器的时候，显示的是页面B。
	
2. android:allowRetainTaskState
	>该属性用来标记是否能够保持原有的状态，但是该属性仅仅只对根Activity起作用（所谓根Activity一般指app的主页面）。
3. android:clearTaskOnLaunch
	>如果该属性为true，那么当启动该Activity时，便会清除该Activity所在任务栈的其他所有Activity。
4. android:finishOnTaskLaunch
	>如果设置该属性为true，那么点击home键回到屏幕之后，再次点击应用图标进入引用之后，系统自动销毁该Activity。
5. android:alwaysRetainTaskState
	>如果当前任务栈的根Activity的该属性设置为true，在该任务栈stop之后仍然保持该任务栈中所有Activity的状态。


# 系统配置变化导致的Activity变化

## 1. Device Configurations
>Orientation, Keyboard, Language.
## 2. 原理简介及解决方案
>一般情况下，当Device Configuration 在Application运行时发生变化，那么系统会自动重启该Activity（此时先onSaveInstance保存数据，然后执行onDestroy,最后执行onCreate）。

>所以我们必须在Activity销毁之前使用onSaveInsance保存数据，在onCreate或者onRestoreInsanceState中回复数据，以此来提供良好的用户体验。但是有时，**我们需要保存大量的数据**，遇到这种情况一般有两种解决方案：<br>


### a. 引用对象

>由于在回调onSaveInstanceState保存的数据不适合保存大批量的数据对象（例如bitmap），而且保存的数据对象必须是Serialized的。这种情况下，当系统配置发送变化时，我们通过引用Fragment来保存数据，在fragment中保存数据对象。具体实现如下：
	
	//注意：当保存数据时，千万不要保存任何引用Activity实例的对象，否则会造成内存泄漏
	

```

	public class RetainedFragment extends Fragment {
	
	    // data object we want to retain
	    private MyDataObject data;
	
	    // this method is only called once for this fragment
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        // retain this fragment
	        setRetainInstance(true);**重点内容**
	    }
	
	    public void setData(MyDataObject data) {
	        this.data = data;
	    }
	
	    public MyDataObject getData() {
	        return data;
	    }
	}
```
	
然后使用FragmentManager将fragment添加到Activity中。

	
```

public class MyActivity extends Activity {
	
	    private RetainedFragment dataFragment;
	
	    @Override
	    public void onCreate(Bundle savedInstanceState) {
	        super.onCreate(savedInstanceState);
	        setContentView(R.layout.main);
	
	        // find the retained fragment on activity restarts
	        FragmentManager fm = getFragmentManager();
	        dataFragment = (DataFragment) fm.findFragmentByTag(“data”);
	
	        // create the fragment and data the first time
	        if (dataFragment == null) {
	            // add the fragment
	            dataFragment = new DataFragment();
	            fm.beginTransaction().add(dataFragment, “data”).commit();
	            // load the data from the web
	            dataFragment.setData(loadMyData());
	        }
	
	        // the data is available in dataFragment.getData()
	        ...
	    }
	
	    @Override
	    public void onDestroy() {
	        super.onDestroy();
	        // store the data in the fragment
	        dataFragment.setData(collectMyLoadedData());
	    }
	}
```


### b. 自己处理系统配置变化引起的改变

 当系统配置发生变化时，如果Activity不需要更新数据或自动应用资源，那么可以声明自己处理该配置变化。

 首先,在AndroidManifest中作如下声明：

	
```

	<activity android:name=".MyActivity"
			  <-- 在此处定义自己要处理的变化-->
	          android:configChanges="orientation|keyboardHidden"
	          android:label="@string/app_name">
	          
```

当在xml中声明的任意配置发生变化时，**系统不会自动重启该Activity**，此时系统将回调onConfigurationChanged()。

> ***注意：*** 当App的targetSdkVersion大于等于13，如果您想处理屏幕方向切换配置变化，那么你必须在android:configuration中包含screenSize属性。


# Activity的各个方法及调用时机


# Fragment的各个方法及调用时机

# FragmentManager

# FragmentPagerAdapter vs FragmentStatePagerAdapter
相信大家都使用过viewPager+PagerAdapter的方法来进行页面切换，其中PageAdapter提供page，viewPager显示page。当page的内容发生变化时，PagerAdapter将此变化通知给ViewPager，也就是说PagerAdapter负责创建Page、管理page，ViewPager负责显示page。但是因为各种各样的需求，要求PagerAdapter提供对page不同的管理方式，比如当page较少时，从page1切换到page2,再切换到page3时，要求不得将page1~page2销毁，这样当用户返回page1时，page1的内容会立马展示给用户；当page较多时，考虑到android系统对单一应用内存的限制，就不可能将所有的page保存起来，接下来我们详细了解一下系统为我们提供的各种PagerAdapter。
## PagerAdapter
该类是page最简单的管理类，如果继承自该类，至少需要实现 instantiateItem(), destroyItem(), getCount() 以及 isViewFromObject()。

>* getItemPosition()
	* 该函数用以返回给定对象的位置，给定对象是由 instantiateItem() 的返回值	
在 ViewPager.dataSetChanged() 中将对该函数的返回值进行判断，以决定是否最终触发 PagerAdapter.instantiateItem() 函数。
在 PagerAdapter 中的实现是直接传回 POSITION_UNCHANGED。如果该函数不被重载，则会一直返回 POSITION_UNCHANGED，从而导致 ViewPager.dataSetChanged() 被调用时，认为不必触发 PagerAdapter.instantiateItem()。很多人因为没有重载该函数，而导致调用
PagerAdapter.notifyDataSetChanged() 后，什么都没有发生。
>* public Object instantiateItem(View container, int position)
	* 在每次 ViewPager 需要一个用以显示的 Object 的时候，该函数都会被 ViewPager.addNewItem() 调用。
>* public void notifyDataSetChanged() 
	* 在数据集发生变化的时候，一般 Activity 会调用 PagerAdapter.notifyDataSetChanged()，以通知 PagerAdapter，而 PagerAdapter 则会通知在自己这里注册过的所有 DataSetObserver。其中之一就是在 ViewPager.setAdapter() 中注册过的 PageObserver。PageObserver 则进而调用 ViewPager.dataSetChanged()，从而导致 ViewPager 开始触发更新其内含 View 的操作。

```

## FragmentPagerAdapter
FragmentPagerAdapter 继承自 PagerAdapter。相比通用的 PagerAdapter，该类更专注于每一页均为 Fragment 的情况。如文档所述，该类内的每一个生成的 Fragment 都将保存在内存之中，因此适用于那些相对静态的页，数量也比较少的那种；如果需要处理有很多页，并且数据动态性较大、占用内存较多的情况，应该使用FragmentStatePagerAdapter。FragmentPagerAdapter 重载实现了几个必须的函数，因此来自 PagerAdapter 的函数，我们只需要实现 getCount()，即可。且，由于 FragmentPagerAdapter.instantiateItem() 的实现中，调用了一个新增的虚函数 getItem()，因此，我们还至少需要实现一个 getItem()。因此，总体上来说，相对于继承自 PagerAdapter，更方便一些。

>* getItem()
	* 该类中新增的一个虚函数。函数的目的为生成新的 Fragment 对象。重载该函数时需要注意这一点。在需要时，该函数将被 instantiateItem() 所调用。如果需要向 Fragment 对象传递相对静态的数据时，我们一般通过 Fragment.setArguments() 来进行，这部分代码应当放到 getItem()。它们只会在新生成 Fragment 对象时执行一遍。如果需要在生成 Fragment 对象后，将数据集里面一些动态的数据传递给该 Fragment，那么，这部分代码不适合放到 getItem() 中。因为当数据集发生变化时，往往对应的 Fragment 已经生成，如果传递数据部分代码放到了 getItem() 中，这部分代码将不会被调用。这也是为什么很多人发现调用 PagerAdapter.notifyDataSetChanged() 后，getItem() 没有被调用的一个原因。
>* instantiateItem()
	* 函数中判断一下要生成的 Fragment 是否已经生成过了，如果生成过了，就使用旧的，旧的将被 Fragment.attach()；如果没有，就调用 getItem() 生成一个新的，新的对象将被 FragmentTransation.add()。
FragmentPagerAdapter 会将所有生成的 Fragment 对象通过 FragmentManager 保存起来备用，以后需要该 Fragment 时，都会从 FragmentManager 读取，而不会再次调用 getItem() 方法。
如果需要在生成 Fragment 对象后，将数据集中的一些数据传递给该 Fragment，这部分代码应该放到这个函数的重载里。在我们继承的子类中，重载该函数，并调用 FragmentPagerAdapter.instantiateItem() 取得该函数返回 Fragment 对象，然后，我们该 Fragment 对象中对应的方法，将数据传递过去，然后返回该对象。
否则，如果将这部分传递数据的代码放到 getItem()中，在 PagerAdapter.notifyDataSetChanged() 后，这部分数据设置代码将不会被调用。
destroyItem()
该函数被调用后，会对 Fragment 进行 FragmentTransaction.detach()。这里不是 remove()，只是 detach()，因此 Fragment 还在 FragmentManager 管理中，Fragment 所占用的资源不会被释放。

## FragmentStatePagerAdapter
FragmentStatePagerAdapter 和前面的 FragmentPagerAdapter 一样，是继承子 PagerAdapter。但是，和 FragmentPagerAdapter 不一样的是，正如其类名中的 'State' 所表明的含义一样，该 PagerAdapter 的实现将只保留当前页面，当页面离开视线后，就会被消除，释放其资源；而在页面需要显示时，生成新的页面(就像 ListView 的实现一样)。这么实现的好处就是当拥有大量的页面时，不必在内存中占用大量的内存。

>* getItem()
	* 一个该类中新增的虚函数。函数的目的为生成新的 Fragment 对象。Fragment.setArguments() 这种只会在新建 Fragment 时执行一次的参数传递代码，可以放在这里。由于 FragmentStatePagerAdapter.instantiateItem() 在大多数情况下，都将调用 getItem() 来生成新的对象，因此如果在该函数中放置与数据集相关的 setter 代码，基本上都可以在 instantiateItem() 被调用时执行，但这和设计意图不符。毕竟还有部分可能是不会调用 getItem() 的。因此这部分代码应该放到 instantiateItem() 中。
>* instantiateItem()
	* 除非碰到 FragmentManager 刚好从 SavedState 中恢复了对应的 Fragment 的情况外，该函数将会调用 getItem() 函数，生成新的 Fragment 对象。新的对象将被 FragmentTransaction.add()。FragmentStatePagerAdapter 就是通过这种方式，每次都创建一个新的 Fragment，而在不用后就立刻释放其资源，来达到节省内存占用的目的的。
>* destroyItem()
	* 将 Fragment 移除，即调用 FragmentTransaction.remove()，并释放其资源。


# Activity, ViewPager, Fragment实现懒加载

# Activity启动与加载过程
  当用户启动Activity时，Instrumentation会接收该请求，然后instrumentation利用Binder向ActivityManagerService发请求。ActivityManagerService内部维护者Activity的调用堆栈（ActivityStack）及各个Activity的状态同步，ActivityManagerService通过ActivityThread去管理Activity的状态从而完成Activity的生命周期的管理。
