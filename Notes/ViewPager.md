# ViewPager

> 此笔记主要是说一下ViewPager如何实现预加载，以及如何实现懒加载、禁止左右滑动。

* ViewPager 作为平时开发时经常使用的控件，我们多是配合TabLayout、嵌套Fragment使用。

## 预加载

* 当ViewPager中嵌套的Fragment多于2个的时候，ViewPager就会预加载当前显示Fragment左右两侧的Fragment。

#### ViewPager是如何实现预加载的？

翻看ViewPager我们可以发现一个常量  *private static final int DEFAULT_OFFSCREEN_PAGES = 1;* 其实，这就是实现ViewPager预加载的值，这个值的意义也是  默认ViewPager当前变量的值为1。

那么ViewPager具体是如何实现的呢？继续翻看源码。

ViewPager源码中全局搜查 *DEFAULT_OFFSCREEN_PAGES*，发现将 *DEFAULT_OFFSCREEN_PAGES* 赋值给了 *mOffscreenPageLimit* ，那我们继续搜查 *mOffscreenPageLimit* ，发现 又赋值给了 *pageLimit* ，好了，我们终于找到了我们要看的逻辑。源码中我们发现，通过当前的Item与pageLimit计算左右需要预加载页面的角标。

计算的方法如下：


```

    final int startPos = Math.max(0, mCurItem - pageLimit);
final int N = mAdapter.getCount();
final int endPos = Math.min(N - 1, mCurItem + pageLimit);
```

好了，我们来模拟一下，假设此时ViewPager中有10个页面，当前页面为3。pageLimit = mOffscreenPageLimit = DEFAULT_OFFSCREEN_PAGES = 1;


```
startPos = Math.max(0 , 3 - 1); = Math.max(0 ,2); = 2;
n = 10;
endPos = Math.min(10 - 1 , 3 + 1); = Math.min(9 , 4); = 4;

也就是说：

startPos = 2;
n = 10;
endPos = 4;

这样就实现了ViewPager的预加载功能。
```

## 懒加载

#### ViewPager如何实现懒加载？

* 上面我们分析了ViewPager是如何实现预加载的，可是有时我们不想实现ViewPager的预加载功能，因为用户可能不会查看预加载的页面就退出了，而且预加载的页面如果有联网操作，也会消耗用户的流量。
* 那么，我们如何实现懒加载呢？

别怕，上面我们既然找到了ViewPager如何实现预加载的方法，我们可以修改 DEFAULT_OFFSCREEN_PAGES = 0 ，使其不实现预加载。我们可以验证一下：

```

    final int startPos = Math.max(0, mCurItem - pageLimit);
final int N = mAdapter.getCount();
final int endPos = Math.min(N - 1, mCurItem + pageLimit);
```

同样，还假设ViewPager中有10个页面，当前页面为第3个。pageLimit = mOffscreenPageLimit = DEFAULT_OFFSCREEN_PAGES = 0;

```
startPos = Math.max(0 , 3 - 0); = Math.max(0 ,3); = 3;
n = 10;
endPos = Math.min(10 - 1 , 3 + 0); = Math.min(9 , 3); = 3;

也就是说：

startPos = 3;
n = 10;
endPos = 3;

这样就实现了ViewPager的懒加载功能。
```

但是 DEFAULT_OFFSCREEN_PAGES =1; 是ViewPager中默认的值，即便是我们打开ViewPager的源码将其修改为0，运行后还是默认为1 ，我们如果要实现懒加载，只有将 ViewPager 下的代码完全复制一份，然后自建一LazyViewPager 继承于ViewGroup，然后将代码粘贴过来，并将 DEFAULT_OFFSCREEN_PAGES 的值修改为0即可，使用的时候，直接使用LazyViewPager即可。

## ViewPager禁止左右滑动

#### ViewPager如何禁止左右滑动？

* 我们在使用ViewPager的时候一般里面嵌套的Fragment会使用ListView或者RecyclerView 亦或者有轮播图，但是在左右滑动的时候是轮播图滑动呢还是让ViewPager左右滑动呢？
* 这个时候我们就要禁止ViewPager的左右滑动操作。我们如何操作呢？

其实禁止ViewPager的左右滑动也很简单，从事件分发机制考虑即可。翻看ViewPager的 onInterceptTouchEvent 方法，也可以看到注释：

```
public boolean onInterceptTouchEvent(MotionEvent ev){
    /*
     * This method JUST determines whether we want to intercept thmotion.
     * If we return true, onMotionEvent will be called and we do thactual
     * scrolling there.
     */
 
    ...
 }
```
什么意思呢？大概意思是说：这个方法决定了我们是否要截断动作。如果返回true,onMotionEvent将被调用，我们在那里执行实际的滚动。

所以，我们可以大概理解为：是否要拦截事件，如果拦截就自己处理，如果不拦截就将事件传递给下面的子View处理。我们可以发现，此方法返回的是boolean类型，所以，我们要禁止左右滑动的话，只需要返回false即可，意思就是不拦截事件，传递给下面的子View。



```
    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }
```


* 但是如果viewpage里面子控件不是viewgroup,还是会调用 onTouchEvent 方法，所以，我们还要处理onTouchEvent这个方法。

查看ViewPager的onTouchEvent发现，喔，里面处理的逻辑好多，看的头蒙，怎么办呢？不用急，简单的说此方法大概意思是：是否自己消费事件。如果自消费，事件就结束。如果不消费就传递给父控件。所以，我们想实现禁止左右滑动，只需要ViewPager不消费事件即可：


```
    @Override
    public boolean onTouchEvent(MotionEvent ev) {
        return false;
    }
```

综上所述，如果我们想实现ViewPager的禁止左右滑动，只要覆写 onInterceptTouchEvent 方法 和 onTouchEvent 方法即可。

* 如果考虑复用，即使用同一个ViewPager ，有时可以左右滑动，有时禁止，那么我们可以写一个方法让其实现是否可以左右滑动。

```
    public void setNoScroll(boolean noScroll) {
        mNoScoll = noScroll;
    }
    
```
然后在 onTouchEvent 方法和 onInterceptTouchEvent方法判断处理即可：


```
 @Override
    public boolean onTouchEvent(MotionEvent ev) {

        if (mNoScoll) {
            return false;
        } else {
            return super.onTouchEvent(ev);
        }


    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (mNoScoll) {
            return false;
        } else {
            return super.onInterceptTouchEvent(ev);
        }
    }
```

简单说一下，mNoScoll 默认false ，如果我们现在的ViewPager 不想实现左右滑动，只需要 setNoScroll(true) 即可，此时mNoScoll 为false，然后在 onTouchEvent 方法和 onInterceptTouchEvent 方法的时候已经 拦截 和 不消费了。

