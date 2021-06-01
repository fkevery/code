---
description: tablayout 使用小细节处理
---

TabLayout 使用时有以下问题需要解决：

# padding

TabLayout 默认情况下每一个 Tab 都会有左右 padding，可在 xml 通过设置 tabPaddingEnd/Start 取消

```xml
app:tabPaddingEnd="0dp"
app:tabPaddingStart="0dp"
```

# TabIndicator(指示器)

## 长短

默认情况下指示器的宽度只有两种形式：
1. 跟内容一样宽。也就是文字有多宽，指示器显示多宽
2. 跟 Tab 一样宽。此时包括设置的 tabPaddingEnd/Start

但**需求指未器的宽度比文字要短**，此时**可以自定义 drawable，可以任意指定宽度**

```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android"
    android:shape="rectangle">

    <item android:gravity="center">
        <shape>
            <size
                android:width="26dp"
                android:height="2dp" />
            <solid android:color="#FE3B30" />
            <corners android:radius="2dp" />

        </shape>
    </item>

</layer-list>
```

## 颜色

虽然可通过 drawable 的 xml 文件指定指示器样式，但**xml 中定义的颜色无效，必须通过代码重新设置**

```kotlin
// 使用 TabLayout 调用
setSelectedTabIndicatorColor(Color.parseColor("#232930"))
// drawable_indicator 就是上面定义的 xml 文件
setSelectedTabIndicator(R.drawable.drawable_indicator)
```

## 圆角

如果指示器要显示圆角，除了在 xml 文件中指定外，还需要额外指定指示器的高度

```java
val d = ResourcesCompat.getDrawable(resources, R.drawable.tab_indicator, null)?.apply {
    setBounds(0, 0, intrinsicWidth, dp2px(2.0f)) // 设置高度
}
// 将 drawable 设置为指示器
setSelectedTabIndicator(d)
```

同时，tablayout 也要加上属性：
```xml
app:tabIndicatorHeight="2dp"
```

# Tab 之间 margin

默认情况下，tab 是没办法指定 margin 的。两个 tab 如果想隔开，就必须指定 padding。若想指定 margin，需要重新定义 TabLayout，如下：

```kotlin
class MarginTabLayout(context: Context, attributeSet: AttributeSet) : TabLayout(context, attributeSet) {

    var tabRightMargin: Int = -1
    // 每生成一个 Tab 时就会调用该方法
    override fun addTab(tab: Tab, position: Int, setSelected: Boolean) {
        super.addTab(tab, position, setSelected)
        addMargin(tab)
    }

    private fun addMargin(tab: Tab) {
        if (tabRightMargin == -1) {
            return
        }
        // tab.view 就是 TabView，它是每一个 Tab 真正显示的 View
        tab.view.apply {
            val layoutParams = this.layoutParams
                    ?: MarginLayoutParams(ViewGroup.LayoutParams.WRAP_CONTENT, ViewGroup.LayoutParams.WRAP_CONTENT)
            if (layoutParams is MarginLayoutParams) {
                layoutParams.rightMargin = tabRightMargin
            }
            this.layoutParams = layoutParams
        }
    }
}
```

