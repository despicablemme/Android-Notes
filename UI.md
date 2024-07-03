# UI

## 问题记录

### button无法修改颜色，总是紫色
在res/values/themes.xml和res/values-night/themes.xml中，原代码为：
```
<style name="Theme.MyApplication" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
```
添加：
```
<style name="Theme.MyApplication" parent="Theme.MaterialComponents.DayNight.DarkActionBar.Bridge">
```

###

## Button设置背景颜色、按下颜色等
在res/drawable目录下新建xml文件：
```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">

        <item android:state_pressed="true"><shape>
            <solid android:color="@android:color/holo_red_light" />
            <stroke android:width="1dp" android:color="@color/material_dynamic_neutral80" />
            <corners android:radius="100dp"/>
        </shape></item>


        <item android:state_enabled="false"><shape>
            <solid android:color="@color/material_dynamic_neutral80" />
            <stroke android:width="1dp" android:color="@color/material_dynamic_neutral80" />
            <corners android:radius="100dp"/>
        </shape></item>

    <item><shape>
        <solid android:color="@android:color/white" />
        <stroke android:width="1dp" android:color="@color/material_dynamic_neutral80" />
        <corners android:radius="100dp"/>
    </shape></item>


</selector>
```
在其中设置一个selector，

## SurfaceView
surfaceView继承自View，可以通过代码改变大小：
```
Holder holder = getSurfaceHolder();
holder.setSizeFromLayout();
holder.setFixedSize(...);
```

## ListView
[ListView文章](https://www.cnblogs.com/lyw-hunnu/p/12687201.html)

1. 首先设置一个ListView；
2. 为ListView中的item设置一个单独的布局xml文件（layout file），其中设定了每个item内的具体布局；
2. 为ListView传入一个Adaptor，可以在Adaptor内设置列表项布局（），点击响应等；
3. Adaptor负责传递数据给ListView的item；
       3.1ArrayAdaptor：可以直接传入一个List，构造adaptor时会将其转化成Array类型；
       3.2 SimpleAdaptor：可以设置更丰富的页面布局，可扩展性好；
```
adapter = new SimpleAdapter(Context context,   // 上下文
                            List<Map<String, Object>>,  // list内每个map对应每个item里要显示的内容，map里存放了多个键值对，键值为名称，对应下面的from参数，值为该控件要显示的东西；
                            R.layout.list_view_layout,  // item的布局
                            String[] from,   // 数据对应的键值，与to一一对应
                            new int[]  // 对应的控件名);
```
       3.3 BaseAdaptor: Adaptor的基础类，可以重写方法自己实现；
4. Adaptor可以通过`setViewBinder`来设定自己的显示逻辑，如如何将data送给view去显示，这样在传给adaptor数据lsit的时候可以更灵活。
```
adapter.setViewBinder(new SimpleAdapter.ViewBinder() {
    @Override
    public boolean setViewValue(View view, Object data, String textRepresentation) {
        ...
        return false;
    }
});
```

## GridView
