layout: hexo
title: 挖VerticalViewPager+RecyclerView滑动冲突的祖坟！
date: 2018-01-03 21:21:22
tags: Android
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>


# 背景
某个阳光明媚到忧伤的午后，突然收到一个诡异的bug提醒，简单明了的写着
**弹幕区域无法滑动！**
一开始无法重现还以为是操作失误之类，后来发现的一个现象让人毛骨悚然:
为啥竖屏下弹幕区域的滑动如丝流畅但横屏下已经成为bug？是卢本伟写的代码么？
![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b79a06f0b932c360e8821)


# 思考过程

复盘这个查bug的过程，大致可分为**定位问题范围**，**定位debug方向**，**关键节点断点调试**，**根据异常信息假设**，**根据正确的假设找根因**。

## 定位问题范围
首先，上来先查看提交日志以及历史构建版本验证，看看是哪个小坏蛋提交的bug呢？一看傻眼，是新版本播放器进行重构时引入的bug，视图层级有了不小的变动：
1.重构前
![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b77f36f0b932c360e881e)

2.重构后
![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b78656f0b932c360e881f)


*ps：这里为了简化问题背景聚焦bug本身，不过多详细描述需求背景，其变更以及实现方式选型，可以后续讨论。这里的VerticalViewpager名如其意，就是竖直方向的Viewpager。*

虽然可以获知是Viewpager+RecyclerView的滑动冲突问题，但目前就只能止步于此，无法更进一步获知具体是哪行代码哪个文件改动产生的bug了。

## 定位debug方向
ok，目前我们知道了Viewpager+RecyclerView的滑动冲突问题，那么.....
并没有什么卵用妈蛋这个不查日志就知道了好么┴┴︵╰（‵□′）╯︵┴┴

冷静冷静，方向很关键，一时找不到方向时，先补充下基础知识开拓思路是很有效的。

### Viewgroup事件分发机制（简略版）

先简单过一遍基础的事件分发机制用到的方法，欲了解详情这里有个博文还不错，可以配合官方文档服用，[Android事件分发机制详解](https://www.jianshu.com/p/38015afcdb58)。


一般而言，我们接触的最多是这三个函数：

- **boolean dispatchTouchEvent (MotionEvent event)**
Pass the touch screen motion event down to the target view, or this view if it is the target.

**Parameters**
*event*	MotionEvent: The motion event to be dispatched.
**Returns**
*boolean*	True if the event was handled by the view, false otherwise.


- **boolean onInterceptTouchEvent (MotionEvent ev)**
Implement this method to intercept all touch screen motion events. This allows you to watch events as they are dispatched to your children, and take ownership of the current gesture at any point.

**Parameters**
*ev*	MotionEvent: The motion event being dispatched down the hierarchy.
**Returns**
*boolean*	Return true to steal motion events from the children and have them dispatched to this ViewGroup through onTouchEvent(). The current target will receive an ACTION_CANCEL event, and no further messages will be delivered here.


- **boolean onTouchEvent (MotionEvent event)**
Implement this method to handle touch screen motion events.

If this method is used to detect click actions, it is recommended that the actions be performed by implementing and calling performClick(). This will ensure consistent system behavior, including:

**Parameters**
*event*	MotionEvent: The motion event.
**Returns**
*boolean*	True if the event was handled, false otherwise.


三者之间的关系可以用一段伪代码描述：
```
// 点击事件产生后，会直接调用dispatchTouchEvent（）方法
public boolean dispatchTouchEvent(MotionEvent ev) {

    //代表是否消耗事件
    boolean consume = false;
  
    
    if (onInterceptTouchEvent(ev)) {
    //如果onInterceptTouchEvent()返回true则代表当前View拦截了点击事件
    //则该点击事件则会交给当前View进行处理
    //即调用onTouchEvent (）方法去处理点击事件
      consume = onTouchEvent (ev) ;

    } else {
      //如果onInterceptTouchEvent()返回false则代表当前View不拦截点击事件
      //则该点击事件则会继续传递给它的子元素
      //子元素的dispatchTouchEvent（）就会被调用，重复上述过程
      //直到点击事件被最终处理为止
      consume = child.dispatchTouchEvent (ev) ;
    }

    return consume;
   }

```

ok，这里我们知道了父ViewGroup最先收到事件并判断是否要拦截事件，如果拦截下来了，由父ViewGroup消耗这个事件，子view就压根收不到事件了。
那么就有同学要问了，那么子view是不是就没有翻身的机会了，根本无法阻止无脑的父ViewGroup拦截自己的事件？
当然不会这么搞啦~！真正的源码里，ViewGroup实现的dispatchTouchEvent里有这么一段代码用来决定是否真的拦截事件：
```
  // Check for interception.
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                // There are no touch targets and this action is not an initial down
                // so this view group continues to intercept touches.
                intercepted = true;
            }
```
重点关注下这行代码
>    final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

这里可以注意到，会通过mGroupFlags里是否有FLAG_DISALLOW_INTERCEPT这个标志位来确定是否要拦截，那么这个标志位什么时候会被设置呢？
动动你们的小手指，alt+f7走起！查找它的引用很快看到：

```
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {

        if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
            // We're already in this state, assume our ancestors are too
            return;
        }

        if (disallowIntercept) {
            mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
        } else {
            mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        }

        // Pass it up to our parent
        if (mParent != null) {
            mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
        }
    }
```
没错，是他，是他，就是他！我们的女神~~三上悠亚~~ **requestDisallowInterceptTouchEvent**！
通过这段代码我们可以知道，子View可以通过requestDisallowInterceptTouchEvent阻止父view拦截事件，从而也能做到某些场景下需要从父ViewGroup里抢事件过来实现功能，常见的就是容器嵌套（Viewpager+RecyclerView，Viewpager+scrollview等等）的滑动冲突问题解决。

好了，到了这里就关键了：
既然工程中用的Viewpager+RecyclerView的手势处理**基本没有更改**的前提下，为什么会存在滑动冲突解决不当的情况？还会有横屏有bug竖屏稳如狗的状况？
![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b7aa46f0b932c360e8822)

# 关键节点断点调试
首先看看作为子ViewGroup的RecyclerView为啥没有成功调用requestDisallowInterceptTouchEvent？
所以先看看作为父ViewGroup的Viewpager在onInterceptTouchEvent里做了什么：

```
 public boolean onInterceptTouchEvent(MotionEvent ev) {
        

      ......
        switch (action) {
            case MotionEvent.ACTION_MOVE: {
                /*
                 * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
                 * whether the user has moved far enough from his original down touch.
                 */

                /*
                * Locally do absolute value. mLastMotionY is set to the y value
                * of the down event.
                */
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }

                final int pointerIndex = ev.findPointerIndex(activePointerId);
                final float x = ev.getX(pointerIndex);
                final float dx = x - mLastMotionX;
                final float xDiff = Math.abs(dx);
                final float y = ev.getY(pointerIndex);
                final float yDiff = Math.abs(y - mInitialMotionY);
                if (DEBUG) Log.v(TAG, "Moved x to " + x + "," + y + " diff=" + xDiff + "," + yDiff);

                if (dx != 0 && !isGutterDrag(mLastMotionX, dx)
                        && canScroll(this, false, (int) dx, (int) x, (int) y)) {
                    // Nested view has scrollable area under this point. Let it be handled there.
                    mLastMotionX = x;
                    mLastMotionY = y;
                    mIsUnableToDrag = true;
                    return false;
                }
                if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff) {
                    if (DEBUG) Log.v(TAG, "Starting drag!");
                    mIsBeingDragged = true;
                    requestParentDisallowInterceptTouchEvent(true);
                    setScrollState(SCROLL_STATE_DRAGGING);
                    mLastMotionX = dx > 0
                            ? mInitialMotionX + mTouchSlop : mInitialMotionX - mTouchSlop;
                    mLastMotionY = y;
                    setScrollingCacheEnabled(true);
                } else if (yDiff > mTouchSlop) {
                    // The finger has moved enough in the vertical
                    // direction to be counted as a drag...  abort
                    // any attempt to drag horizontally, to work correctly
                    // with children that have scrolling containers.
                    if (DEBUG) Log.v(TAG, "Starting unable to drag!");
                    mIsUnableToDrag = true;
                }
                if (mIsBeingDragged) {
                    // Scroll to follow the motion event
                    if (performDrag(x)) {
                        ViewCompat.postInvalidateOnAnimation(this);
                    }
                }
                break;
            }

         ······
        return mIsBeingDragged;
    }
```

可以看到mIsBeingDragged作为是否要拦截事件的标志，而设置mIsBeingDragged的关键在于
```
if (xDiff > mTouchSlop && xDiff * 0.5f > yDiff)
```

这个mTouchSlop是啥呢？就是系统能识别出被认为是滑动的最小距离，小于这个常量，系统不认为你在进行滑动。我们查找它的引用可以看到是由ViewConfigration设定的mPagingTouchSlop：
```
/**
*   Viewpager.java
**/
  final ViewConfiguration configuration = ViewConfiguration.get(context);
  final float density = context.getResources().getDisplayMetrics().density;

//注意这里取的是configuration里的PagingTouchSlop，也就是下面代码里的mPagingTouchSlop
  mTouchSlop = configuration.getScaledPagingTouchSlop();


/**
*  ViewConfiguration.java
**/


 mTouchSlop = res.getDimensionPixelSize(
                com.android.internal.R.dimen.config_viewConfigurationTouchSlop);
 mPagingTouchSlop = mTouchSlop * 2;
        
    
```
跑到R.dimen.config_viewConfigurationTouchSlop来看看具体是什么值
```
  <!-- Base "touch slop" value used by ViewConfiguration as a
         movement threshold where scrolling should begin. -->
    <dimen name="config_viewConfigurationTouchSlop">8dp</dimen>

```
也就是说最小触摸距离为2*8=16dp呗, 在我的工程机（乐视X9000+，464ppi，也就是每平方厘米有190个像素）上就是56px。
那是不是这里的问题呢？带着一丝神奇的第六感，打断点看看：
![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b437c6f0b932c360e8462)

**卧槽，还真是，配合日志可以发现几乎每次滑动，尽管滑动幅度再小，都大于56px，这样的话必然导致Viewpager在onInterceptTouchEvent就拦截下来没有机会给子view接受触摸事件！**

![](https://yunpan.oa.tencent.com/note/api/file/getImage?fileId=5a4b7b4d6f0b932c360e8823)

# 根据异常信息假设
也就是说，可以先做个假设，在横屏的滑动情况下由于某种原因，导致最小的滑动的距离都大于16dp，从而导致作为子View的RecyclerView没办法接受事件。
验证这种想法当然简单，反射修改最小滑动距离！随便搞个大的值看看

```
public VerticalViewpager(Context context) {

		this(context, null);
		init();
	}

	public VerticalViewpager(Context context, AttributeSet attrs) {

		super(context, attrs);

		init();
	}

	private void init() {
    

		try {
			Field slopFiled = Viewpager.class.getDeclaredField("mTouchSlop");
			slopFiled.setAccessible(true);
			slopFiled.set(this,200);
		}catch (Exception e){
			e.printStackTrace();
		}

	}
```

编译运行！

淦！that's it!
![](https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1514957079331&di=07f19205b5da8ae0b9a294dad20ab197&imgtype=0&src=http%3A%2F%2Fphotocdn.sohu.com%2F20160301%2Fmp61143404_1456776129029_2.gif)

# 根据正确的假设找根因
好了激动完了，该找找根因了。
这时候冷静下来想起之前说的一个小细节：
>既然工程中用的Viewpager+RecyclerView的手势处理**基本没有更改**的前提下，为什么会存在滑动冲突解决不当的情况？还会有横屏有bug竖屏稳如狗的状况？

ok,是基本没有更改，但改了哪些呢？

```



public class VerticalViewpager  extends BaseViewpager {

	private static final String TAG ="VerticalViewpager" ;


	public VerticalViewpager(Context context) {

		this(context, null);
		init();
	}

	public VerticalViewpager(Context context, AttributeSet attrs) {

		super(context, attrs);

		init();
	}

	private void init() {
		// The majority of the magic happens here
		setPageTransformer(true, new VerticalPageTransformer());
		// The easiest way to get rid of the overscroll drawing that happens on the left and right
		setOverScrollMode(OVER_SCROLL_NEVER);
	}

	private class VerticalPageTransformer implements Viewpager.PageTransformer {

		@Override
		public void transformPage(View view, float position) {

			if (position < -1) { // [-Infinity,-1)
				// This page is way off-screen to the left.
				view.setAlpha(0);

			} else if (position <= 1) { // [-1,1]
				view.setAlpha(1);

				// Counteract the default slide transition
				view.setTranslationX(view.getWidth() * -position);

				//set Y position to swipe in from top
				float yPosition = position * view.getHeight();
				view.setTranslationY(yPosition);

			} else { // (1,+Infinity]
				// This page is way off-screen to the right.
				view.setAlpha(0);
			}
		}
	}

	/**
	 * Swaps the X and Y coordinates of your touch event.
	 */
	private MotionEvent swapXY(MotionEvent ev) {
		float width = getWidth();
		float height = getHeight();

		float newX = (ev.getY() / height) * width;
		float newY = (ev.getX() / width) * height;

		ev.setLocation(newX, newY);

		return ev;
	}

	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev){
		boolean intercepted = super.onInterceptTouchEvent(swapXY(ev));
		swapXY(ev); // return touch coordinates to original reference frame for any child views
		return intercepted;
	}

	@Override
	public boolean onTouchEvent(MotionEvent ev) {
		return super.onTouchEvent(swapXY(ev));
	}

}

```

首先我把目光就落在了swapXY这个方法身上，如果不出bug的话一般也看不出啥毛病的，但第二次看来突然发觉哪有什么不对：

```
/**
	 * Swaps the X and Y coordinates of your touch event.
	 */
	private MotionEvent swapXY(MotionEvent ev) {
		float width = getWidth();
		float height = getHeight();

		float newX = (ev.getY() / height) * width;
		float newY = (ev.getX() / width) * height;

		ev.setLocation(newX, newY);

		return ev;
	}
```
先简化成数学方式比较好描述：

$$记 初始坐标为(X_0,Y_0), 通过swap计算得到的坐标为(X_0^`,Y_0^`),\frac{width}{height}为r,用户触摸屏幕到另一个位置后的坐标为(X_1,Y_1),通过swap方法计算得到的坐标为$$
由此：
$$swap(X_0,Y_0) = (X_0^`,Y_0^`) = (Y_0*r,\frac{X_0}{r})$$
这时注意，VerticalViewpager在onInterceptTouchEvent和onTouchEvent都是调用的super方法，也就是说我们需要关注的是:
$$\nabla{X}=X_1^`-X_0^`=(Y_1-Y_0)*r$$
这样看就很清晰了，**屏幕的长宽比会影响到差值，从而放大或者缩小差值造成误差**！
具体到我们的用的乐视手机上，r=0.5625(竖屏，横屏为1.78）,最小滑动距离56px在竖屏下成了31.5px，横屏下成了99.68px（我神一般的金手指居然还能滑出60px也是厉害了），这就能解释为什么为啥竖屏下弹幕区域的滑动如丝流畅但横屏下已经成为bug了！

# 写在最后
OK，写到这已经把这个bug的祖坟都挖出来了，解决方案也可以出来了
1. 简单粗暴的在播放场景下用反射改变mTouchSlop
2. 重写一遍VerticalViewpager，不要涉及到坐标的转换

显然第一个方案十分粗暴，但第二个方案也是让人倒吸一口凉气，网上虽说也有不少代码，但经历过这一遭后，就算要用也得从头到尾过一遍了，搞不好又会写成另一个文章的历程。但是嘛，这才是保障代码质量的重要步骤，不能成天只当救火员啊！(๑•̀ㅂ•́)و✧

你们可能知道很久的小tips：
1.文中用到的数学公式是由MathJax提供，使用方便简洁，markdown里直接用清爽无比
2.云盘笔记里写的md笔记以及上传的图片，km上均可显示

