
大家好，我是开源图片编辑器的 [https://github.com/ikuaitu/vue\-fabric\-editor](https://github.com) 的作者，它是一款基于 PC 版本的开源图片编辑器。


最近很多开发者咨询，**是否可以将开源图片编辑器改造为一款适用于移动端的 H5 版本图片编辑器**，最近 H5 版本的图片编辑器刚刚上线，就将实现思路和产品细节整理成笔记分享出来，供大家参考。
![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093315116-2078043386.png)


# 基础


开源的图片编辑器的基本功能都有了，例如切换模板、添加元素、自定义字体等，不过相较于移动端的交互会有很大的差异，做了很多改造，这次笔记主要分享一下**移动端图片编辑器实现思路和细节**。
![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093325900-1215290872.png)
![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093334935-1838246478.png)
![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093344301-1539822477.png)


# 大纲


1. 切换模板
2. 添加图片
3. 添加组合元素
4. 设置背景色
5. 修改画布尺寸
6. 快捷菜单
7. 属性工具条
8. 特效字体
9. 切换字体
10. 输入文字
11. 文字排版
12. 边框
13. 阴影
14. 下载图片



> 注：部分代码示例为封装后的代码，非 fabric.js 原生方法。


### 1\. 切换模板


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093413609-187947723.gif)


编辑器基于 fabric.js 开发，所有的模板都是以 json 的格式存储，切换模板只需要请求详情接口，将 json 格式的数据调添加到画布当中即可，需要注意的点是需要将模板中使用的字体名称，并加载字体文件后再进行渲染，否则字体样式没办法正常渲染。



```
const loadInfo = async (res: any) => {
  const info = res.data
  templName.value = info.name;
  await canvasEditor.getFontList(JSON.stringify(info.json));
  canvasEditor.loadJSON(JSON.stringify(info.json), () => LoadingPlugin(false));
};


```

### 2\. 添加图片


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093437162-2089676435.gif)


fabric.js 中添加图片提供了很多种方法，我们使用通过最简单的`fabric.Image.fromURL`即可，另外，经常有图片尺寸大于画布的情况，还需要将图片按画布宽度的一般进行缩放，更方便用户操作。



```
const toEditor = async (e: MouseEvent) => {
  visible.value = false
  LoadingPlugin(true)
  const item = await canvasEditor.createImgByElement(e.target as HTMLImageElement)
  await canvasEditor.addBaseType(item, { scale: true })
  LoadingPlugin(false)
}

```

### 3\. 添加组合元素


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093446971-111124697.gif)


fabric.js 支持将单个元素按照 JSON 格式导出/导入，我们将导出的数据存储在数据库中的，导入时按元素类型导入即可，需要获取 JSON 中元素的类型，并作为方法名调用，同样需要在导入前做字体加载，倒入后做缩放。



```
const capitalizeFirstLetter = (str: string) => {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

const toEditor = async (item: ItemProps) => {
  visible.value = false
  LoadingPlugin(true)
  await canvasEditor.downFontByJSON(JSON.stringify(item.json));
  const el = JSON.parse(JSON.stringify(item.json));
  const elType = capitalizeFirstLetter(el.type);
  new fabric[elType].fromObject(el, (fabricEl: fabric.Object) => {
    canvasEditor.dragAddItem(fabricEl);
    LoadingPlugin(false)
  });
}

```

### 4\. 设置背景色


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093455578-999105103.gif)


设置背景色较为简单，按照 fabric.js 的 API 设置颜色即可，需要注意的是大部分 PC 端的颜色组件并不适配移动端 H5 的场景，不支持 touch 事件，我们使用了 `@jaames/iro`这个组件，它在移动端表现出色，完全适配我们的场景，而且它的 API 很灵活，我们将它封装成一个通用的颜色组件，在多处调用。



```

  
  




```

### 5\. 修改画布尺寸


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093521004-1395170393.gif)


日常使用图片编辑器都有修改画布尺寸的需要，在开源项目中已经封装好了相应的方法，直接调用即可，需要注意的是，当修改尺寸弹框弹出时，为了达到所见即所得的效果，要避免弹框遮挡画布，其他属性修改同理。



```

const resizeEditor = async () => {
    await nextTick()
    const editorWorkspase = document.querySelector('#workspace') as HTMLElement
    const popElement = document.querySelector('.my-editor-popup') as HTMLElement
    const headerElement = document.querySelector('.t-navbar') as HTMLElement
    if (popElement) {
      editorWorkspase.style.height = `calc(100vh - ${popElement?.offsetHeight + headerElement?.offsetHeight || 0}px)`
    } else {
      editorWorkspase.style.height = ''
    }
  }


```

### 6\. 快捷菜单


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093545612-85251011.gif)


很多快捷操作需要能够让用户快速找的并完成操作，我们为元素添加了快捷菜单功能，避免让一些简单的操作让用户在底部菜单栏点来点去，当选中元素时自动展示，取消选中时隐藏即可，需要注意的是在快捷菜单并不总是在元素上方，快捷菜单应该根据元素位置和画布的尺寸进行定位，当菜单超出画布区域时我们要及时调整菜单位置；另外 当属性弹框出现，画布尺寸变化时，需要同步修改菜单位置。



```
// 更新位置信息
const upDatePosition = async () => {
  const activeObject = canvasEditor.canvas.getActiveObject();
  if (activeObject) {
    canvasEditor.canvas.renderAll();
    fixLeft.value = 10;
    fixTop.value = 10;
    await nextTick();
    isIncluded(activeObject);
    await nextTick();
  }
}

// 监听选中对象变化更新位置信息
getObjectAttr(upDatePosition)
canvasEditor.canvas.on('selection:updated', upDatePosition)
canvasEditor.canvas.on('mouse:move', upDatePosition)
canvasEditor.on('workspaceAutoEvent', upDatePosition)

```

### 7\. 属性工具条


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093557354-722804612.gif)


参考了其他图片编辑器，部分属性在点击元素后才会出现可修改选项，取消选中时便隐藏选项，另外 选中的元素不同，可修改选项也不同，这是一个在移动端做复杂图片编辑器中非常棒的一个交互。


我们封装了通用的选中类型和方法，针对每个属性组件单独设置隐藏/展示。


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093605673-794554959.png)


### 8\. 特效字体


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093613570-1618007989.gif)


特效字体主要是文字元素的颜色、边框、阴影的组合，我们将来文字设置样式后的 JSON 导出并保存在数据库中，当选中某一个特效时，将属性按 JSON 中的数据设置给元素即可。



```
const setStyle = (item: ImgItem) => {
  const activeObject = canvasEditor.canvas.getActiveObjects()[0];
  if (activeObject) {
    const values = toRaw(item.json);
    const keys = ['fill', 'stroke', 'strokeWidth', 'shadow', 'strokeLineCap'];
    activeObject.set('paintFirst', 'stroke');
    keys.forEach((key) => {
      activeObject.set(key, values[key]);
      if (key === 'fill' && typeof values[key] != 'string') {
        activeObject.set(key, new fabric.Gradient(values[key]));
      }
    });
    canvasEditor.canvas.renderAll();
  }
};

```

### 9\. 切换字体


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093624619-236577360.gif)


修改字体只需要调用 fabric.js 元素的`fontFamily`属性即可，在修改之前要确保字体加载完成。



```

const changeCommon = async (key: string, value: any) => {
  const activeObject = canvasEditor.canvas.getActiveObjects()[0];
  if (activeObject) {
    LoadingPlugin(true);
    baseAttr.fontFamily = value;
    try {
      await canvasEditor.loadFont(value)
    } catch (error) {
      console.log(error)
    }
    LoadingPlugin(false);
    activeObject && activeObject.set(key, value);
    canvasEditor.canvas.renderAll();
  }
};

```

### 10\. 输入文字


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093635675-1695722703.gif)


fabric.js 可直接双击文字元素进行修改，不过在移动端这种交互并不醒目，我们单独为文本元素进行了修改，选中元素后，再次点击时弹出输入框，可以在底部菜单栏点击按钮进行修改。


### 11\. 文字排版


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093646694-1688086487.gif)


文字排版较为简单，我们只需要按照 fabric.js 的文字属性对文字进行属性设置即可，如 fontSize、lineHeight、charSpacing 等。



```
// 属性值
const baseAttr = reactive({

  fontSize: 0,
  lineHeight: 0,
  charSpacing: 0,
  textAlign: '',

  fontWeight: '',
  fontStyle: '',

  underline: false,
  linethrough: false,
  overline: false,
});



```

### 12\. 边框


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093655025-825576973.gif)


边框样式和文字样式类似，配合颜色组件可以很快捷的实现功能。



```
// 属性值
const baseAttr = reactive({
  stroke: '#fff',
  strokeWidth: 0,
  strokeDashArray: [],
});


```

### 13\. 阴影


![](https://img2024.cnblogs.com/blog/693044/202409/693044-20240923093706887-44761403.gif)


引用属性主要是元素的 shadow 子属性的修改，代码如下：



```
// 属性值
const baseAttr = reactive({
  shadow: {
    color: '#fff',
    blur: 0,
    offsetX: 1,
    offsetY: 1,
  }
});


// 通用属性改变
const changeCommon = () => {
  const activeObject = canvasEditor.canvas.getActiveObjects()[0];
  if (activeObject) {
    activeObject.set('shadow', new fabric.Shadow(baseAttr.shadow));
    canvasEditor.canvas.renderAll();
  }
};

```

### 14\. 下载图片


fabric.js 可以导出 Png/Jpeg/Base64 格式的图片，同时 JPEG 格式还可以指定图片质量与尺寸倍数，详见 fabric.js 的 API 文档。


# 结尾


以上就是 fabric.js 开发移动端编辑器的实现细节了，结合我们的开源项目和插件化架构可以很方便的完成项目开发，如果你在做类似项目或者做类似的项目，欢迎与我交流。


开源项目：[https://github.com/ikuaitu/vue\-fabric\-editor/blob/main/README\-zh.md](https://github.com):[veee加速器](https://liuyunzhuge.com)


