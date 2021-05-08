---
description: 用代码实现 drawable
---

项目中经常需要定义各种 drawable，有时候就因为修改一个颜色就需要重新定义一个 xml 文件，太麻烦。解决办法就是**用代码生成各种 drawable**。

可以将 drawable 分为两种维度，每一个 drawable 的最终生成都是由两个维度配合完成：

1. 内容：实心或空心
2. 形状：椭圆/圆、矩形、圆角矩形

代码如下：

```kotlin
object DrawableEx {
    /**
     * 实心椭圆/圆
     * **如果 View 的宽高相同，就会是一个圆**
     */
    fun solidOval(color: Int): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(color = color)
        // 每一个 drawable 就是内容与形状组合完成，因此从枚举 content 中取一个，从 Shape 中取一个，两者依次处理
        // 最终得到 drawable
        arrayOf<DrawableHandler>(DrawableHandler.Content.SOLID, DrawableHandler.Shape.OVAL).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }

    /**
     * 空心椭圆/圆。**如果 View 的宽高相同，就会是一个圆**
     * [stroke] 单位 px
     */
    fun strokeOval(color: Int, stroke: Int): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(strokeColor = color, stroke = stroke)
        arrayOf<DrawableHandler>(DrawableHandler.Content.STROKE, DrawableHandler.Shape.OVAL).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }

    /**
     * 实心矩形
     */
    fun solidRect(color: Int): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(color = color)
        arrayOf<DrawableHandler>(DrawableHandler.Content.SOLID, DrawableHandler.Shape.RECT).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }

    /**
     * 空心矩形
     * [stroke] 单位 px
     */
    fun strokeRect(strokeColor: Int, stroke: Int = 1): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(strokeColor = strokeColor, stroke = stroke)
        arrayOf<DrawableHandler>(DrawableHandler.Content.STROKE, DrawableHandler.Shape.RECT).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }

    /**
     * 圆角实心矩形
     * [radius] 圆角半径，单位 px
     */
    fun roundSolidRect(color: Int, radius: Float): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(color = color, cornerRadius = radius)
        arrayOf<DrawableHandler>(DrawableHandler.Content.SOLID, DrawableHandler.Shape.ROUND).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }

    /**
     * 圆角空心矩形
     * [stroke] 单位 px
     * [radius] 圆角半径，单位 px
     */
    fun roundStrokeRect(color: Int, stroke: Int, radius: Float): Drawable {
        val drawable = GradientDrawable()
        val config = DrawableConfig(stroke = stroke, strokeColor = color, cornerRadius = radius)
        arrayOf<DrawableHandler>(DrawableHandler.Content.STROKE, DrawableHandler.Shape.ROUND).forEach {
            it.handle(drawable, config)
        }
        return drawable
    }
    // 定义不同的枚举需要的参数封装类
    class DrawableConfig(val color: Int = Color.TRANSPARENT, val stroke: Int = 0, val strokeColor: Int = Color.TRANSPARENT, val cornerRadius: Float = 0f)

    // 每一个枚举都要实现的接口
    interface DrawableHandler {
        fun handle(drawable: GradientDrawable, config: DrawableConfig)
        // 实心空心
        enum class Content : DrawableHandler {
            SOLID {
                override fun handle(drawable: GradientDrawable, config: DrawableConfig) {
                    drawable.setColor(config.color)
                }
            },
            STROKE {
                override fun handle(drawable: GradientDrawable, config: DrawableConfig) {
                    drawable.setStroke(config.stroke, config.strokeColor)
                }

            }
        }
        // 形状
        enum class Shape : DrawableHandler {

            RECT {
                override fun handle(drawable: GradientDrawable, config: DrawableConfig) {
                    drawable.shape = GradientDrawable.RECTANGLE
                    drawable.cornerRadius = 0f
                }
            },

            ROUND {
                override fun handle(drawable: GradientDrawable, config: DrawableConfig) {
                    drawable.shape = GradientDrawable.RECTANGLE
                    drawable.cornerRadius = config.cornerRadius
                }
            },
            OVAL {
                override fun handle(drawable: GradientDrawable, config: DrawableConfig) {
                    drawable.shape = GradientDrawable.OVAL
                }
            }
        }
    }
}
```


