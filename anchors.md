# anchors note

> Anchors example for mxGraph. This example demonstrates defining 
> fixed connection points for all shapes.
>
> mxGraph的锚点demo。演示为所有形状定义固定结点。

## knowledge

### 常见的 `terminal` 是什么？

`terminal` n. 航空站；终点站；终端机；线接头；末端；晚期病人

这个单词我一直的理解就是终端工具，英文中的涵义即末端。这个词会在官方API中常看到，一般来讲都是出现在与edge有关的场景，特指edge的两端，即结点。

### 全局默认锚点的设置方法

```javascript
mxShape.prototype.constraints = [
  new mxConnectionConstraint(new mxPoint(0.25, 0), true),
  // ...
];
```

要点：
1. `constraints` 中的元子即节点上会生成的锚点。
2. mxPoint是一个二维坐标，**(0, 0)** 即指节点的左上角，**(1, 1)** 即右下角。
3. `mxConnectionConstraint` 第二个参数 `perimeter` 是指定是否要求锚点位于节点的周长上。
4. 当mxPoint的区间超出(0, 1)，且 `perimeter` 为 *true* 时，坐标轴的位置会异常，因为不会这样去写代码，也就没细研究mxPoint的实现。

### 重置mxPolyline的constraints

demo在 `mxShape.prototype.constraint` 设置值后，紧接着执行了

```javascript
// Edges have no connection points
mxPolyline.prototype.constraints = null;
```

`mxPolyline` 是一种多折点的线，是edge，它继承自 `mxShape`，因为线是没有锚点的，所以当对默认锚点设置值后，为了不影响 `mxPolyline`，所以要有此设置。

### mxEvent.disableContextMenu

Disables the context menu for the given element.

这里可以考虑一下我们能自定义一些右键上下文弹窗样式。

### 拖拽线时线的预览样式设置

```javascript
// Enables connect preview for the default edge style
graph.connectionHandler.createEdgeState = function (me) {
  var edge = graph.createEdge(null, null, null, null, null);
  return new mxCellState(this.graph.view, edge, this.graph.getCellStyle(edge));
```

参考 [一掘金文章](https://juejin.im/post/5d2eaa9e6fb9a07f04207cca#heading-8)中时发现了关于这段代码的解释，这样设置以后，拖拽线时的预览也是拆线的了，会与最后生成的线样式一致。

## question

### demo中关于 `getAllConnectionConstraints` 的重写意义何在？

demo开篇即重写：

```javascript
// Overridden to define per-shape connection points
mxGraph.prototype.getAllConnectionConstraints = function (terminal, source) {
  if (terminal != null && terminal.shape != null) {
    if (terminal.shape.stencil != null) {
      return terminal.shape.stencil.constraints;
    } else if (terminal.shape.constraints != null) {
      return terminal.shape.constraints;
    }
  }
  return null;
};
```

而源码中关于 `getAllConnectionConstraints` 的源码为：

```javascript
mxGraph.prototype.getAllConnectionConstraints = function(terminal, source) {
  if (terminal != null && terminal.shape != null && terminal.shape.stencil != null) {
    return terminal.shape.stencil.constraints;
  }
  return null;
};
```

逻辑完全一致，那干嘛还要重写？

### graph.setConnectable

指定画布是否允许创建新的链接。这个使用的场景不太明白，可能是某些场景需要禁止画布操作吧。

## link

* [example/anchors.html](https://github.com/jgraph/mxgraph/blob/master/javascript/examples/anchors.html)
* [mxGraph.getAllConnectionConstraints](http://jgraph.github.io/mxgraph/docs/js-api/files/view/mxGraph-js.html#mxGraph.getAllConnectionConstraints)
*[mxConnectionConstraint](http://jgraph.github.io/mxgraph/docs/js-api/files/view/mxConnectionConstraint-js.html#mxConnectionConstraint)
* [mxPolyline](http://jgraph.github.io/mxgraph/docs/js-api/files/shape/mxPolyline-js.html#mxPolyline)
* [MDN contextmenu_event](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/contextmenu_event)
* [mxEvent.disableContextMenu](http://jgraph.github.io/mxgraph/docs/js-api/files/util/mxEvent-js.html#mxEvent.disableContextMenu)
* [mxGraph 入门实例教程](https://juejin.im/post/5d2eaa9e6fb9a07f04207cca)