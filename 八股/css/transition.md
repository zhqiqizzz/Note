`transition` 就是让 CSS 属性变化时“慢慢变”，而不是瞬间切换。它适合做按钮悬停、卡片浮起、菜单展开、颜色变化这类简单动画。

**基础例子**

```
<button class="btn">提交</button>
```

```
.btn {
  background: #2563eb;
  color: white;
  padding: 10px 18px;
  border: 0;
  border-radius: 6px;
  cursor: pointer;

  transition: background 0.3s ease;
}

.btn:hover {
  background: #1d4ed8;
}
```

解释：

```
transition: background 0.3s ease;
```

意思是：

```
当 background 发生变化时
用 0.3 秒完成变化
变化速度曲线是 ease
```

没有 `transition` 时，颜色会瞬间变深。加上后，颜色会平滑过渡。

**多个属性一起过渡**

```
<div class="card">
  商品卡片
</div>
```

```
.card {
  width: 180px;
  padding: 24px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: white;

  transition: transform 0.25s ease, box-shadow 0.25s ease;
}

.card:hover {
  transform: translateY(-6px);
  box-shadow: 0 10px 24px rgba(0, 0, 0, 0.16);
}
```

鼠标移上去时：

```
卡片向上移动
阴影变明显
两个变化都在 0.25 秒内完成
```

这里推荐用 `transform`，因为它比改 `top`、`margin-top` 更适合动画，性能更好。

**transition 的完整写法**

```
.box {
  transition-property: width;
  transition-duration: 0.3s;
  transition-timing-function: ease;
  transition-delay: 0.1s;
}
```

等价简写：

```
.box {
  transition: width 0.3s ease 0.1s;
}
```

四个部分分别是：

```
transition-property：要过渡的属性
transition-duration：持续时间
transition-timing-function：速度曲线
transition-delay：延迟多久开始
```

**常见速度曲线**

```
.box {
  transition: transform 0.3s linear;
}
```

```
linear：匀速
ease：先慢后快再慢，默认值
ease-in：开始慢，后面快
ease-out：开始快，后面慢
ease-in-out：开始和结束都慢
```

实际 UI 里，常用：

```
transition: transform 0.2s ease, opacity 0.2s ease;
```

**延迟过渡**

```
<div class="tip">提示内容</div>
```

```
.tip {
  opacity: 0;
  transform: translateY(8px);
  transition: opacity 0.2s ease 0.3s, transform 0.2s ease 0.3s;
}

.tip.show {
  opacity: 1;
  transform: translateY(0);
}
```

这里 `0.3s` 是 delay，表示等 0.3 秒后再开始动画。

**不是所有属性都适合 transition**

适合过渡的属性：

```
color
background-color
opacity
transform
width / height
margin / padding
border-color
box-shadow
```

不适合或不能平滑过渡的属性：

```
display
position
font-family
background-image
```

比如这个不会平滑：

```
.menu {
  display: none;
  transition: display 0.3s ease;
}

.menu.show {
  display: block;
}
```

`display` 只能瞬间切换。

可以改用：

```
.menu {
  opacity: 0;
  transform: translateY(-8px);
  pointer-events: none;
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.menu.show {
  opacity: 1;
  transform: translateY(0);
  pointer-events: auto;
}
```

**常见展开示例**

```
<button class="toggle">菜单</button>
<div class="panel">
  <p>首页</p>
  <p>设置</p>
  <p>退出</p>
</div>
```

```
.panel {
  max-height: 0;
  overflow: hidden;
  opacity: 0;
  transition: max-height 0.3s ease, opacity 0.2s ease;
}

.toggle:hover + .panel {
  max-height: 160px;
  opacity: 1;
}
```

为什么用 `max-height`？

因为 `height: auto` 不能直接过渡：

```
height: 0;
height: auto;
```

浏览器不知道从具体多少过渡到 `auto`，所以不会平滑。

**重点记忆**

```
transition: 属性名 持续时间 速度曲线 延迟时间;
```

最常用写法：

```
transition: all 0.3s ease;
```

但进阶后更推荐写具体属性：

```
transition: transform 0.3s ease, opacity 0.3s ease;
```

因为 `all` 可能把不需要动画的属性也加进去，带来性能问题或奇怪效果。

一句话总结：

```
transition 负责“状态变化之间的平滑过程”。
默认状态写 transition，变化状态比如 :hover、.active 里写最终样式。
```