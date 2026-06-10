`animation` 是 CSS 里更完整的动画系统。和 `transition` 不同，`transition` 需要“状态变化”触发，比如 `:hover`；而 `animation` 可以自己按关键帧运行。

简单说：

```
transition：从 A 状态过渡到 B 状态
animation：按照关键帧自己播放一段动画
```

**1. 最基础例子：盒子左右移动**

```
<div class="box"></div>
```

```
.box {
  width: 80px;
  height: 80px;
  background: #2563eb;
  animation: move 2s ease-in-out infinite;
}

@keyframes move {
  from {
    transform: translateX(0);
  }

  to {
    transform: translateX(200px);
  }
}
```

解释：

```
animation: move 2s ease-in-out infinite;
```

意思是：

```
使用 move 这段关键帧动画
持续 2 秒
速度曲线 ease-in-out
无限循环 infinite
```

`@keyframes move` 定义动画过程：

```
from：开始状态
to：结束状态
```

**2. 使用百分比关键帧**

如果动画不只是起点和终点，可以用百分比。

```
<div class="ball"></div>
```

```
.ball {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  background: #ef4444;
  animation: bounce 1.2s ease-in-out infinite;
}

@keyframes bounce {
  0% {
    transform: translateY(0);
  }

  50% {
    transform: translateY(-80px);
  }

  100% {
    transform: translateY(0);
  }
}
```

效果：

```
小球从原位向上跳
到最高点
再回到原位
不断循环
```

**3. animation 的完整属性**

完整写法：

```
.box {
  animation-name: move;
  animation-duration: 2s;
  animation-timing-function: ease;
  animation-delay: 0.5s;
  animation-iteration-count: infinite;
  animation-direction: alternate;
  animation-fill-mode: both;
  animation-play-state: running;
}
```

简写：

```
.box {
  animation: move 2s ease 0.5s infinite alternate both;
}
```

每个属性含义：

```
animation-name：动画名称，对应 @keyframes
animation-duration：动画持续时间
animation-timing-function：速度曲线
animation-delay：延迟多久开始
animation-iteration-count：播放次数
animation-direction：播放方向
animation-fill-mode：动画前后状态
animation-play-state：播放或暂停
```

**4. 播放次数 animation-iteration-count**

播放一次：

```
.box {
  animation: move 2s ease 1;
}
```

播放三次：

```
.box {
  animation: move 2s ease 3;
}
```

无限循环：

```
.box {
  animation: move 2s ease infinite;
}
```

**5. 播放方向 animation-direction**

```
.box {
  animation: move 2s ease infinite alternate;
}
```

常见值：

```
normal：正常，从 0% 到 100%
reverse：反向，从 100% 到 0%
alternate：奇数次正向，偶数次反向
alternate-reverse：奇数次反向，偶数次正向
```

`alternate` 很常用，比如左右摇摆、呼吸效果。

```
.box {
  animation: swing 1s ease-in-out infinite alternate;
}

@keyframes swing {
  from {
    transform: rotate(-8deg);
  }

  to {
    transform: rotate(8deg);
  }
}
```

**6. 动画结束后停在哪里 animation-fill-mode**

默认情况下，动画结束后元素会回到原始样式。

```
.box {
  animation: fadeIn 1s ease;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }

  to {
    opacity: 1;
  }
}
```

如果元素原本没有 `opacity: 1`，动画结束后可能回到原样。

常用：

```
.box {
  animation: fadeIn 1s ease forwards;
}
```

`forwards` 表示动画结束后停在最后一帧。

常见值：

```
none：默认，不保留动画状态
forwards：结束后保留最后一帧
backwards：延迟期间应用第一帧
both：同时具备 forwards 和 backwards
```

例子：

```
.card {
  opacity: 0;
  transform: translateY(20px);
  animation: enter 0.4s ease forwards;
}

@keyframes enter {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

卡片从下方淡入，结束后保持可见。

**7. 暂停和播放 animation-play-state**

```
<div class="loader"></div>
```

```
.loader {
  width: 48px;
  height: 48px;
  border: 4px solid #ddd;
  border-top-color: #2563eb;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.loader:hover {
  animation-play-state: paused;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
```

鼠标移上去时，加载动画暂停。

**8. 多个动画同时使用**

```
.box {
  animation:
    move 2s ease infinite alternate,
    colorChange 2s linear infinite;
}

@keyframes move {
  from {
    transform: translateX(0);
  }

  to {
    transform: translateX(200px);
  }
}

@keyframes colorChange {
  from {
    background: #2563eb;
  }

  to {
    background: #ef4444;
  }
}
```

多个动画用逗号分隔。

**9. 常见加载动画**

```
<div class="spinner"></div>
```

```
.spinner {
  width: 40px;
  height: 40px;
  border: 4px solid #e5e7eb;
  border-top-color: #2563eb;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
```

原理：

```
圆形边框
顶部边框换颜色
整体不断旋转
看起来像 loading
```

**10. 呼吸灯动画**

```
<div class="dot"></div>
```

```
.dot {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: #22c55e;
  animation: pulse 1.4s ease-in-out infinite;
}

@keyframes pulse {
  0% {
    transform: scale(1);
    opacity: 1;
  }

  50% {
    transform: scale(1.5);
    opacity: 0.45;
  }

  100% {
    transform: scale(1);
    opacity: 1;
  }
}
```

适合在线状态、提醒状态。

**11. 骨架屏动画**

```
<div class="skeleton"></div>
```

```
.skeleton {
  width: 240px;
  height: 24px;
  border-radius: 4px;
  background: linear-gradient(
    90deg,
    #e5e7eb 25%,
    #f3f4f6 37%,
    #e5e7eb 63%
  );
  background-size: 400% 100%;
  animation: shimmer 1.4s ease infinite;
}

@keyframes shimmer {
  0% {
    background-position: 100% 0;
  }

  100% {
    background-position: 0 0;
  }
}
```

原理：

```
背景是横向渐变
通过 background-position 移动渐变
产生扫光效果
```

**12. animation 和 transition 的区别**

```
transition：
- 需要状态变化触发
- 通常只有开始和结束
- 适合 hover、active、展开收起

animation：
- 可以自动播放
- 可以有多个关键帧
- 可以循环、暂停、反向播放
- 适合 loading、复杂动效、入场动画
```

例子：

```
.button {
  transition: background 0.2s ease;
}

.button:hover {
  background: red;
}
```

这是 transition。

```
.loader {
  animation: spin 1s linear infinite;
}
```

这是 animation。

**13. 性能建议**

优先动画这些属性：

```
transform
opacity
```

谨慎动画这些属性：

```
width
height
top
left
margin
padding
```

因为它们可能触发布局重排，动画容易卡。

推荐：

```
.box {
  transform: translateX(100px);
}
```

少用：

```
.box {
  left: 100px;
}
```

**14. 最常用模板**

入场动画：

```
.card {
  opacity: 0;
  transform: translateY(16px);
  animation: fadeUp 0.35s ease forwards;
}

@keyframes fadeUp {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

循环旋转：

```
.icon {
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
```

左右浮动：

```
.float {
  animation: float 2s ease-in-out infinite alternate;
}

@keyframes float {
  from {
    transform: translateY(0);
  }

  to {
    transform: translateY(-12px);
  }
}
```

一句话总结：

```
animation = @keyframes 定义过程 + animation 应用到元素。
```

你只要记住这个结构：

```
.box {
  animation: 动画名 持续时间 速度曲线 播放次数 方向 填充模式;
}

@keyframes 动画名 {
  0% {}
  50% {}
  100% {}
}
```

就能写出大部分 CSS 动画。