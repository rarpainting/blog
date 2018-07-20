# flex布局兼容性问题

## 推荐网站
### http://expo.bootcss.com/

## 一.W3C各个版本的flex
### 2009 version
标志：display: box; or a property that is box-{*} (eg. box-pack)

### 2011 version
标志：display: flexbox; or the flex() function or flex-pack property

### 2012 version
标志：display: flex/inline-flex; and flex-{*} properties

### 2014 version
新增了对flex项z-index的规定

### 2015 W3C Editor’s Draft
没有大的改动

P.S.注意2015的是W3C Editor’s Draft，只是个草案，还处于修修改改的阶段，还没有征集浏览器供应商的意见

## 二.浏览器兼容性
关于flex的W3C规范： http://dev.w3.org/csswg/css-flexbox-1/

浏览器兼容性可以参考CanIUse： http://caniuse.com/#feat=flexbox

根据CanIUse的数据可以总结如下：

- IE10部分支持2012，需要-ms-前缀
- Android4.1/4.2-4.3部分支持2009 ，需要-webkit-前缀
- Safari7/7.1/8部分支持2012， 需要-webkit-前缀
- IOS Safari7.0-7.1/8.1-8.3部分支持2012，需要-webkit-前缀

所以需要考虑新版本2012： http://www.w3.org/TR/2012/CR-css3-flexbox-20120918/
而Android需要考虑旧版本2009： http://www.w3.org/TR/2009/WD-css3-flexbox-20090723/

## 三.浏览器兼容的flex语法
上面分析得很清楚，针对需要兼容的目标使用对应版本的语法就好了，下面给出常用的布局代码：
```css
/* 子元素-平均分栏 */
.flex1 {
  -webkit-box-flex: 1;      /* OLD - iOS 6-, Safari 3.1-6 */
  -moz-box-flex: 1;         /* OLD - Firefox 19- */
  width: 20%;               /* For old syntax, otherwise collapses. */
  -webkit-flex: 1;          /* Chrome */
  -ms-flex: 1;              /* IE 10 */
  flex: 1;                  /* NEW, Spec - Opera 12.1, Firefox 20+ */
}
/* 父元素-横向排列（主轴） */
.flex-h {
  display: box;              /* OLD - Android 4.4- */
  display: -webkit-box;      /* OLD - iOS 6-, Safari 3.1-6 */
  display: -moz-box;         /* OLD - Firefox 19- (buggy but mostly works) */
  display: -ms-flexbox;      /* TWEENER - IE 10 */
  display: -webkit-flex;     /* NEW - Chrome */
  display: flex;             /* NEW, Spec - Opera 12.1, Firefox 20+ */
  /* 09版 */
  -webkit-box-orient: horizontal;
  /* 12版 */
  -webkit-flex-direction: row;
  -moz-flex-direction: row;
  -ms-flex-direction: row;
  -o-flex-direction: row;
  flex-direction: row;
}
/* 父元素-横向换行 */
.flex-hw {
  /* 09版 */
  /*-webkit-box-lines: multiple;*/
  /* 12版 */
  -webkit-flex-wrap: wrap;
  -moz-flex-wrap: wrap;
  -ms-flex-wrap: wrap;
  -o-flex-wrap: wrap;
  flex-wrap: wrap;
}
/* 父元素-水平居中（主轴是横向才生效） */
.flex-hc {
  /* 09版 */
  -webkit-box-pack: center;
  /* 12版 */
  -webkit-justify-content: center;
  -moz-justify-content: center;
  -ms-justify-content: center;
  -o-justify-content: center;
  justify-content: center;
  /* 其它取值如下：
    align-items     主轴原点方向对齐
    flex-end        主轴延伸方向对齐
    space-between   等间距排列，首尾不留白
    space-around    等间距排列，首尾留白
   */
}
/* 父元素-纵向排列（主轴） */
.flex-v {
  display: box;              /* OLD - Android 4.4- */
  display: -webkit-box;      /* OLD - iOS 6-, Safari 3.1-6 */
  display: -moz-box;         /* OLD - Firefox 19- (buggy but mostly works) */
  display: -ms-flexbox;      /* TWEENER - IE 10 */
  display: -webkit-flex;     /* NEW - Chrome */
  display: flex;             /* NEW, Spec - Opera 12.1, Firefox 20+ */
  /* 09版 */
  -webkit-box-orient: vertical;
  /* 12版 */
  -webkit-flex-direction: column;
  -moz-flex-direction: column;
  -ms-flex-direction: column;
  -o-flex-direction: column;
  flex-direction: column;
}
/* 父元素-纵向换行 */
.flex-vw {
  /* 09版 */
  /*-webkit-box-lines: multiple;*/
  /* 12版 */
  -webkit-flex-wrap: wrap;
  -moz-flex-wrap: wrap;
  -ms-flex-wrap: wrap;
  -o-flex-wrap: wrap;
  flex-wrap: wrap;
}
/* 父元素-竖直居中（主轴是横向才生效） */
.flex-vc {
  /* 09版 */
  -webkit-box-align: center;
  /* 12版 */
  -webkit-align-items: center;
  -moz-align-items: center;
  -ms-align-items: center;
  -o-align-items: center;
  align-items: center;
}
/* 子元素-显示在从左向右（从上向下）第1个位置，用于改变源文档顺序显示 */
.flex-1 {
  -webkit-box-ordinal-group: 1;   /* OLD - iOS 6-, Safari 3.1-6 */
  -moz-box-ordinal-group: 1;      /* OLD - Firefox 19- */
  -ms-flex-order: 1;              /* TWEENER - IE 10 */
  -webkit-order: 1;               /* NEW - Chrome */
  order: 1;                       /* NEW, Spec - Opera 12.1, Firefox 20+ */
}
/* 子元素-显示在从左向右（从上向下）第2个位置，用于改变源文档顺序显示 */
.flex-2 {
  -webkit-box-ordinal-group: 2;   /* OLD - iOS 6-, Safari 3.1-6 */
  -moz-box-ordinal-group: 2;      /* OLD - Firefox 19- */
  -ms-flex-order: 2;              /* TWEENER - IE 10 */
  -webkit-order: 2;               /* NEW - Chrome */
  order: 2;                       /* NEW, Spec - Opera 12.1, Firefox 20+ */
}
```

为了更好的兼容性，我们需要给容器声明flex-h/flex-v，而不是一般的flex：
```css
/* 父元素-flex容器 */
.flex {
  display: box;              /* OLD - Android 4.4- */
  display: -webkit-box;      /* OLD - iOS 6-, Safari 3.1-6 */
  display: -moz-box;         /* OLD - Firefox 19- (buggy but mostly works) */
  display: -ms-flexbox;      /* TWEENER - IE 10 */
  display: -webkit-flex;     /* NEW - Chrome */
  display: flex;             /* NEW, Spec - Opera 12.1, Firefox 20+ */
}
```
所以，建议在需要兼容Android时（2009版语法）采用flex-h/flex-v声明容器使用flex模式，在不需要兼容Android时（2012版语法）使用flex设置容器
注意：上面给的代码并不是完全兼容各个高端浏览器的，但要比任何其它现有代码兼容性好，具体兼容性测试结果请看Demo

## 四.flex布局Demo

在线测试：Demo
测试结果：

- Android4.2.2不支持换行

- Android4.2.2下伪元素位置表现不一致

- IOS Safari 7.1的表现与Chrome28、Chrome43、Firefox的表现一致

更多测试结果请反馈给我，谢谢

注意 ：从测试结果可以发现flex布局会把伪元素当作元素来分配空间（文档好像有提到给伪元素设置position: fixed/absolute;可以避免这一情况，本文暂未验证），但我们一般希望伪元素只有装饰作用，不影响布局，这与我们的预期不一致。所以， **当flex布局中有伪元素时要特别小心，尽可能多地进行浏览器兼容性测试，或者改用float布局**
