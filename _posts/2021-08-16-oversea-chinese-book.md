---
layout: post
title: 中文绘本
subtitle: 中文绘本周报
image: /img/lifedoc/jijiji.jpg
tags: [周报]
---

### 0816 中文绘本

首页目前已经告一段落，下一步是setting界面。

setting界面有很多是可以复用的布局，因此准备使用自定义view去做，自定义view需要使用merge标签，需要在style文件中定义对应布局的style标签。

test

```java

    // 设置顶部栏padding核心就是paddingTop加上statusBar高度，我这里为了这个加了一层布局
    public static void setPaddingSmart(Context context, View view) {
        if (Build.VERSION.SDK_INT >= MIN_API) {
            ViewGroup.LayoutParams lp = view.getLayoutParams();
            if (lp != null && lp.height > 0) {
                lp.height += getStatusBarHeight(context);//增高
            }
            view.setPadding(view.getPaddingLeft(), view.getPaddingTop() + getStatusBarHeight(context),
                    view.getPaddingRight(), view.getPaddingBottom());
        }
    }

        public static int getStatusBarHeight(Context context) {
        int result = 24;
        int resId = context.getResources().getIdentifier("status_bar_height", "dimen", "android");
        if (resId > 0) {
            result = context.getResources().getDimensionPixelSize(resId);
        } else {
            result = (int) TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP,
                    result, Resources.getSystem().getDisplayMetrics());
        }
        return result;
    }
```

