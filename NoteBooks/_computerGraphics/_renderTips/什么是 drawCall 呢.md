每次CPU准备数据并通知GPU的过程就称之为一个DrawCall
在没有任何优化的情况下，每帧绘制 n 个对象，就需要 n 次 drawCall
每帧 drawCall 次数越多，自然渲染的性能越低