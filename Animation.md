# 动画
[动画讲解大全](https://github.com/OCNYang/Android-Animation-Set/tree/master/transition-animation)

## 几个动画关键字
- alpha：透明度；
- translationX(Y):移动；
- scale：缩放；
- rotate：旋转；
## 帧动画
通过xml定义帧，文件为res/animation/下面的frames.xml；
```
<animation-List>
...
<\animation-list>
```
将帧设置为布局的background：
```
android:background = "@animation/frames"
```
代码中启动：
```
AnimationDrawable animation = (AnimationDrawable) imageView.getBackground();
animation.start();
animation.stop;
```

## 补间动画
### 插值器Interpolator
按照时间走过的比例计算当前移动的百分比来控制变化速度；比如一共十秒，走过2s，那么估值器的
百分比为20%（线性），使用其他估值或自定义估值时按公式计算；比如加速插值器，实现加速需要随时间
变化比例的速率逐渐提升，返回一个比例；

插值器反映随时间变化，数据变化的速度，给出当前时间的变化距离（所走的比例）；
- 得到相对位置

### 估值器TypeEvaluator
估值器通过当前的速度来计算当前的值，如果仅仅数学计算的话用默认即可，比如涉及特殊类型的值，可以进行自定义，
如颜色变化，计算rgb值；
- 得到绝对位置与一些特殊的值计算

### xml写法
```
<alpha xmlns:android="http://schemas.android.com/apk/res/android"  
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"  
    android:fromAlpha="1.0"  
    android:toAlpha="0.1"  
    android:duration="2000"/>

<scale xmlns:android="http://schemas.android.com/apk/res/android"  
        android:interpolator="@android:anim/accelerate_interpolator"  
        android:fromXScale="0.2"  
        android:toXScale="1.5"  
        android:fromYScale="0.2"  
        android:toYScale="1.5"  
        android:pivotX="50%"     // 距离左上角起点的比例来确定中心点（50%为图中点）
        android:pivotY="50%"  
        android:duration="2000"/>

<translate xmlns:android="http://schemas.android.com/apk/res/android"  
    android:interpolator="@android:anim/accelerate_decelerate_interpolator"  
    android:fromXDelta="0"  
    android:toXDelta="320"  
    android:fromYDelta="0"  
    android:toYDelta="0"  
    android:duration="2000"/>

<rotate xmlns:android="http://schemas.android.com/apk/res/android"  
        android:interpolator="@android:anim/accelerate_decelerate_interpolator"  
        android:fromDegrees="0"  
        android:toDegrees="360"  
        android:duration="1000"  
        android:repeatCount="1"  
        android:repeatMode="reverse"/>

// 动画集合
<set
interpolator = "">
  <scale>
  <alpha>
</set>
```
加载定义的动画,然后用要动画的view启动它：
```
Animation animation = AnimationUtils.loadAnimation(this, R.animation.$file_name);
view.startAnimation(animation);
```
实现动画状态监听：
```
animation.setAnimationListener(new AnimationListener{
  @Onverride onAnimationStart(){}
  @Override onAnimationRepeat(){}
  @Override onAnimationEnd(){}
  });
```

### 动态写法
代码创建一个animation对象，然后startAnimation();

### 为Fragment设置动画
调用`FragmentTransaction`对象的`setTransition(int transit);`;

自定义：`setCustomAnimations(int enter, exit, popEnter, popExit);`

### 为activity设置动画
在startActivity(intent)或finish()后添加：
`overrideActivityTransition(int enterAnim, int exitAnim)`

## 属性动画(Property Animator)
改变控件的属性，来实现；

### ValueAnimator
通过一个ValueAnimator的`ofInt/ofFloat(...)`方法创建实例，传入动画过程中的目标值，
接着在update回调中修改目标属性：
```
animator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener(){
    @Override onAnimatorUpdate(ValueAnimator animation){
      int value = (int) animation.getAnimationValue;
     }
  })
```
设置插值器，然后启动即可：
```
animator.setIterpolator(...);
animator.start();
```

### ObjectAnimator
传入要改变的属性值，通过属性自带的get...和set...方法来获取和修改，后面的int传入规则看方法注释；
```
ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f, 1f);
animator.setDuration(5000);
animator.start();
```

### 组合AnimatorSet
传入objectAnimator或者ValueAnimator
```
AnimatorSet animSet = new AnimatorSet();
animSet.play(rotate).with(fadeInOut).after(moveIn);
animSet.setDuration(5000);
animSet.start();
```

### 状态监听与实现
上面三种ValueAnimator、ObjectAnimator、AnimatorSet都可以使用`addListner()`:
```
anim.addListener(new AnimatorListener() {
	@Override public void onAnimationStart(Animator animation) {}

	@Override public void onAnimationRepeat(Animator animation) {}

	@Override public void onAnimationEnd(Animator animation) {}

	@Override public void onAnimationCancel(Animator animation) {}
});
```
### xml实现属性动画
参数写入xml，可以组合成set:
```
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:ordering="sequentially" >

    <objectAnimator>
    <set android:ordering="together" >
        <objectAnimator>
        <set android:ordering="sequentially" >
            <objectAnimator>
            <objectAnimator>
        </set>
    </set>
</set>
```
加载：
```
Animator animator = AnimatorInflater.loadAnimator(context, R.animator.anim_file);
animator.setTarget(view);
animator.start();
```

## Transition
实现过场动画，如activity、fragment的动画切换；
### xml实现
