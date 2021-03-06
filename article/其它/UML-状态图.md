# UML —— 状态图



> 状态图描述了一个状态机，状态机由状态、转换、事件、活动和动作组成。

## 创建UML图

![image-20210328090533566](https://azmddy.github.io/img/image-20210328090533566.png)

## 元素

+ 初始状态（Initial State）:每个状态图都一个初始状态，用实心圆表示，表示状态机的开始，一个状态图只能有一个初始状态。

![image-20210328093906045](https://azmddy.github.io/img/image-20210328093906045.png)

+ 状态（State）：描述对象在某个时期处于的状态，用圆角矩形表示，状态描述包括`名称`、`进入动作(Entry Activity)`、`退出动作(Exit Activity)`、`执行动作(Do Activity)`以及`约束(Constraint)`。 状态又可分为`组合状态(Composite State`和`简单状态(Simple State)`，简单状态不可分，不包含其他状态，而组合状态是指内部嵌套有其他子状态的状态。

![image-20210328101612370](https://azmddy.github.io/img/image-20210328101612370.png)

+ 转换：用带箭头的直线表示。一端连接上一个状态，一端连接下一个目标状态，可以标注相关转换信息（事件，活动，动作）。
  + 事件：指引起状态发生变化的事件；
  + 条件：当接受到事件时，判断条件，满足条件就进行激活转换；
  + 动作：一个可执行的原子操作；
+ 终止状态（Final State）：一个状态图可以拥有一个或多个终止状态，用含有实心圆的空心圆表示。
+ 判定：根据条件进行判定，在状态图中使用空心菱形表示。

