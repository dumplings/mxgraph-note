# boundary note

> Boundary example for mxGraph. This example demonstrates implementing boundary events in bpmn diagrams.
> mxGraph的边界、范围示例。演示了在BPMN中边界事件的实现。

## Knowledge Points

### What is Boundary

`Boundary`, n. 边界；范围；分界线。

这个例子可以应用到一个复杂带有分支概念的复合性节点中。

### mxGraph中的Style

样式的学习，官方支持很少，包括有一些我在代码中看到他们的设置值，官方的API中连枚举都不给出，看源码也一时不好查到。只能说看到的尽量记一下。需要有效的文档支持。

不过就概念而言，可以参考一下 [引导](https://jgraph.github.io/mxgraph/docs/manual.html#3.1.3.1) 文档中的简述。

### `mxGraph.prototype.isCellFoldable` 返回指定对象是否可折叠

```javascript
/**
 * Function: isCellFoldable
 * 
 * Returns true if the given cell is foldable. This implementation
 * returns true if the cell has at least one child and its style
 * does not specify <mxConstants.STYLE_FOLDABLE> to be 0.
 * 
 * Parameters:
 * 
 * cell - <mxCell> whose foldable state should be returned.
 */
mxGraph.prototype.isCellFoldable = function(cell, collapse) {
  var state = this.view.getState(cell);
  var style = (state != null) ? state.style : this.getCellStyle(cell);

  return this.model.getChildCount(cell) > 0 && style[mxConstants.STYLE_FOLDABLE] != 0;
};
```

上面是官方的源码，参数中的 `collapse` 在源码中未使用，同时Parameters中也未解释。不过在源码中多处调用该函数的地方看到有传值。
值为 `isCellCollapsed` 的反值。应该是为了给改写行为提供的吧，不过API文档确实差一些啊。

源码的逻辑是当给定的cell的子节点数大于0，并且cell的 `STYLE_FOLDABLE` 非0时，即返回 `true`。

案例中对源码进行了改写：

```javascript
graph.isCellFoldable = function(cell, collapse) {
  var childCount = this.model.getChildCount(cell);
  for (var i = 0; i < childCount; i++) {
    var child = this.model.getChildAt(cell, i);
    var geo = this.getCellGeometry(child);

    if (geo != null && geo.relative) {
      return false;
    }
  } 
  return childCount > 0;
}
```

由上面的改写，我得出以下几点：

* `this.model.getChildCount` 可使用 `cell.getChildCount` 来代替，简写代码；
* `this.model.getChildAt` 可使用 `cell.getChildAt` 来代替，简写代码；
* `this.getCellGeometry` 可使用 `cell.getGeometry` 或 `cell.geometry` 来代替，不过这个也要看 `this.getCellGeometry` 是否已经被改写；
* 改写方法当遍历子节点时发现任一子节点如果几何信息中的 `relative` 值为 `true` 时，则返回 `false`，告之当前节点组不可折叠；
* 该demo的逻辑应该是通过设置 `relative` 来为组进行折叠阻止行为；

### 自定义函数 `getRelativePosition`

函数实现：

```javascript
/**
 * @params state: 被移动的cell的state
 * @params dx: x轴移动位移
 * @params dy: y轴移动位移
 */
function getRelativePosition(state, dx, dy) {
  if (state != null) {
    var model = graph.getModel();
    var geo = model.getGeometry(state.cell);
    if (geo != null && geo.relative && !model.isEdge(state.cell)) {
      var parent = model.getParent(state.cell);
      if (model.isVertex(parent)) {
        var pstate = graph.view.getState(parent);
        if (pstate != null) {
          var scale = graph.view.scale;
          /**
           * state中的x/y应该是相对于根层的坐标值，而geometry中的x/y是相对于父容器的向量偏移。
           */
          var x = state.x + dx;
          var y = state.y + dy;
          if (geo.offset != null) {
            x -= geo.offset.x * scale;
            y -= geo.offset.y * scale;
          }
          /**
           * 下面的x和y，算出的是占比，也是二维向量。
           * 吐槽一下官方写的代码再定义一个x和y不行吗，一定要在原xy上改吗，这样代码很不明确。
           */
          x = (x - pstate.x) / pstate.width;
          y = (y - pstate.y) / pstate.height;
          // todo 这个判断条件的逻辑没有看懂，只是知道他的判断结果为点是在Y轴上还是X轴上
          if (Math.abs(y - 0.5) <= Math.abs((x - 0.5) / 2)) {
            // 说明在Y轴上
            x = (x > 0.5) ? 1 : 0;
            y = Math.min(1, Math.max(0, y));
          } else {
            // 说明在Y轴上
            x = Math.min(1, Math.max(0, x));
            y = (y > 0.5) ? 1 : 0;
          }
          return new mxPoint(x, y);
        }
      }
    }
  }
  return null;
};
```

要了解该函数的意义，首先要掌握什么是[mxPoint](https://jgraph.github.io/mxgraph/docs/js-api/files/util/mxPoint-js.html#mxPoint)。

> Implements a 2-dimensional vector with double precision coordinates.
> 实现具有双精度坐标的二维向量。

该函数得出的即子节点相对于父容器的二维向量形式的坐标值。

// todo 待补充

### graph的 `translateCell` 函数

通过给定的cell进行坐标的转变，函数的调用发生在 `mxGraph.prototype.cellsMoved` 中，即当cell发生位移时，便会调用该函数。

首先看一下改写后的源码：

```javascript
graph.translateCell = function (cell, dx, dy) {
  var rel = getRelativePosition(this.view.getState(cell), dx * graph.view.scale, dy * graph.view.scale);
  if (rel != null) {
    var geo = this.model.getGeometry(cell);
    if (geo != null && geo.relative) {
      geo = geo.clone();
      geo.x = rel.x;
      geo.y = rel.y;
      this.model.setGeometry(cell, geo);
    }
  } else {
    mxGraph.prototype.translateCell.apply(this, arguments);
  }
};
```

首先通过getRe

## Questions

## Links

* [mxConstants.STYLE_xxx](https://jgraph.github.io/mxgraph/docs/manual.html#3.1.3.1)
* [mxPoint](https://jgraph.github.io/mxgraph/docs/js-api/files/util/mxPoint-js.html#mxPoint)