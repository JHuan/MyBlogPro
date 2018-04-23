layout: hexo
title: 你用的那个Sort还是当初你认识的那个sort么？
date: 2017-08-03 20:51:12
tags: Android
---

## 问题
正值明日之子大直播火热之际，好不容易在一个周六可以在吃着法国蜗牛的我生平第一次收到了rdm的crash告警，每隔5分钟就发到微信上的那种，那种看到我小心肝不止的在颤抖的那种告警。
赶紧看看是什么crash突然在如此重要的直播里crash了，一眼看去居然是个从未见过的crash。还好堆栈明了可以马上知道是道具弹幕中排序 的锅，但这是个什么鬼？

```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
java.util.ComparableTimSort.mergeHi(ComparableTimSort.java:848)
java.util.ComparableTimSort.mergeAt(ComparableTimSort.java:466)
java.util.ComparableTimSort.mergeCollapse(ComparableTimSort.java:389)
java.util.ComparableTimSort.sort(ComparableTimSort.java:178)
java.util.ComparableTimSort.sort(ComparableTimSort.java:142)
java.util.Arrays.sort(Arrays.java:1957)
java.util.Collections.sort(Collections.java:1877)
com.tencent.qqlivebroadcast.business.bulletscreen.presenter.GiftDanmakuPresenter.void onGetPushDanmaku(java.util.List)(GiftDanmakuPresenter.java:150)
...

```
赶紧问谷歌爸爸，初步结论是：**由于jdk7和Android采用的是排序方法是TimSort，它对于元素之间等价关系的校验更为严格，如果违反了等价关系则有可能出发崩溃。**
看完居然是依旧懵逼的。

OK，咱们一个个来，复习基础知识，探究下为什么会有这样的问题呢？里面到底隐藏了什么大~~咪咪~~秘密？

## 等价关系你还记得么？
设R是集合A上的一个二元关系，若R满足：
- **自反性**：∀a∈A,=>(a,a)∈R
- **对称性**：(a,b)∈R∧a≠b=>(b,a)∈R
- **传递性**：(a,b)∈R,(b,c)∈R=>(a,c)∈R
则称R是定义在A上的一个等价关系。effective java中第三章。有对应的关于对象关于compareTo的建议规范讲到这个这三个属性

## TimSort是个什么鬼？那个叫Tim的同学你放学后别走
先来弄明白TimSort是什么排序先，由于摘自维基百科：
> Timsort是结合了合并排序（merge sort）和插入排序（insertion sort）而得出的排序算法，它在现实中有很好的效率。Tim Peters在2002年设计了该算法并在Python中使用（TimSort 是 python 中 list.sort 的默认实现）。该算法找到数据中已经排好序的块-分区，每一个分区叫一个run，然后按规则合并这些run。Pyhton自从2.3版以来一直采用Timsort算法排序，现在Java SE7和Android也采用Timsort算法对数组排序。

核心思路过程：
>TimSort 算法为了减少对升序部分的回溯和对降序部分的性能倒退，将输入按其升序和降序特点进行了分区。排序的输入的单位不是一个个单独的数字，而是一个个的块-分区。其中每一个分区叫一个run。针对这些 run 序列，每次拿一个 run 出来按规则进行合并。每次合并会将两个 run合并成一个 run。合并的结果保存到栈中。合并直到消耗掉所有的 run，这时将栈上剩余的 run合并到只剩一个 run 为止。这时这个仅剩的 run 便是排好序的结果。
综上述过程，Timsort算法的过程包括
（0）如何数组长度小于某个值，直接用二分插入排序算法
（1）找到各个run，并入栈
（2）按规则合并run

具体的算法实现过程感兴趣的同学可以直接阅读源码，也有相关的博客解读和分析这个挺复杂且其实有bug的算法。这里不做过多阐述因为实在是...有够繁琐的。
这里直接讲会触发这crash的校验代码:
```
       if (len2 == 1) {
            if (DEBUG) assert len1 > 0;
            dest -= len1;
            cursor1 -= len1;
            System.arraycopy(a, cursor1 + 1, a, dest + 1, len1);
            a[dest] = tmp[cursor2];  // Move first elt of run2 to front of merge
        } else if (len2 == 0) {
            throw new IllegalArgumentException(
                "Comparison method violates its general contract!");
        } else {
            if (DEBUG) assert len1 == 0;
            if (DEBUG) assert len2 > 0;
            System.arraycopy(tmp, 0, a, dest - (len2 - 1), len2);
        }
```
此举是在两个run归并之前做个的一个优化，假设已有两个有序的runA和runB，归并开始前会把其中一个run中需要合并的数据拷贝到tmp数组中（这里假设将runB中的部分数据拷入）。每次归并比较时会记录其中一个run和tmp段比较“胜利（大于对方）”的次数，为了得知runA是否平均大于runB，如果达到最大次数，会有一次tmp数据数据直接拷贝到runA中的操作，拷贝完并做一次上述代码的校验维护算法的正确性。



## 为啥传统的归并排序没有这种问题？

最简单最单纯的归并排序过程：
    1、Divide: 把长度为n的输入序列分成两个长度为n/2的子序列。
    2、Conquer: 对这两个子序列分别采用归并排序。
    3、Combine: 将两个排序好的子序列合并成一个最终的排序序列。

盗个图说明下归并过程：
![归并排序过程](http://km.oa.com/files/photos/pictures/201708/1501667820_89_w554_h542.jpeg)

可以看到，归并排序并不校验元素的等价关系，只是仅仅判断两个元素之间的大小就可以完成归并。

## 直接原因？
### 业务场景
腾讯直播对于道具弹幕会有个排序需求（大土豪送的跑车游艇当然要优先展示啊~），目前来说需求就很简单，送的道具价值越大越靠前，由后台下的一个字段giftPriority来做排序关键字，让前端可以将获得的弹幕队列进行排序。

### 排序实现过程
由于历史原因，目前有两个通道下发道具弹幕信息，分别为push通道和轮询通道（简称poll通道)，分别实现为PushGiftDanmakuContent和GiftDanmakUContent,继承于BaseDanmakuContent。

于是要做排序，那就需要用支持弹幕的compare啦，直接上代码：

```
/**
 * 普通弹幕内容类
 * Created by barryjiang on 2016/7/8.
 */
public abstract class BaseDanmakuContent implements Comparable<BaseDanmakuContent> {

    //弹幕类型
    protected DanmakuContentType    mCommentType;

    //昵称
    protected String mSenderNickName = "";

    //内容
    protected String mContent = "";

    //存入的时间
    protected long currentTime;


    @Override
    public int compareTo(@NonNull BaseDanmakuContent another) {
        //按时间排序，先存入的排在前面，FIFO
        return (int) (((BaseDanmakuContent) another).currentTime - currentTime );

    }

  
}


/**
 * poll弹幕道具弹幕内容
 * Created by barryjiang on 2016/7/8.
 */
public class GiftDanmakuContent extends BaseDanmakuContent {

    private GiftInfo giftInfo;

    public GiftInfo getGiftInfo() {
        return giftInfo;
    }

    public void setGiftInfo(GiftInfo giftInfo) {
        this.giftInfo = giftInfo;
    }
 
}


/**
 * 通过push通道下发的弹幕内容
 * Created by barryjiang on 2017/4/13.
 */

public class PushGiftDanmakuContent extends BaseDanmakuContent {

    //优先级，越高优先级越高，用于弹幕排序
    private int                     giftPriority = 0;

    @Override
    public int compareTo(@NonNull BaseDanmakuContent another) {
        if(another instanceof PushGiftDanmakuContent){
            PushGiftDanmakuContent pushGiftDanmakuContent = (PushGiftDanmakuContent) another;

            //先按priority排，最后按存入的时间排
            if(priority - pushGiftDanmakuContent.getGiftPriority() !=0 ){
                return priority - pushGiftDanmakuContent.getGiftPriority();
            }else {
                return super.compareTo(another);
            }

        }else {
            return super.compareTo(another);
        }

    }
   
}


```
相信聪明的读者已经看出了问题了，这样的实现违反了等价关系中的传递性。可以容易证明存在pushDanmakuA>pushDanmakuB,pushDanmakuB>PollDanmakuC,PollDanmakuC>pushDanmakuA这样的异常。

而且测试没能复现也正是因为要触发TimSort的归并排序之前，如果元素数目没有达到最小归并数，则采用二分插入排序，一样不会校验等价关系。

## 后话
排序已经是非常古典的一个算法问题了，也没有想到后续的优化和创新有这么一个坑，但是说到底还是平时编程时的思维习惯不够好造成的。愿各位引以为戒。
水平有限，关于算法实现细节的解析可能有偏差也望各位指正。

# 相关阅读：
[OpenJDK 源代码阅读之 TimSort](http://blog.csdn.net/on_1y/article/details/30109975)
[如何找出Timsort算法和玉兔月球车中的Bug？](http://www.freebuf.com/vuls/62129.html)
  








