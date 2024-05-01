---
title: "Rigidbody记录"
categories: 知识点
tags: Unity
---

# ForceMode

ForceMode为枚举类型，用来控制力的作用方式，假设刚体质量为m=2.0f，力向量为f=(10.0f,0.0f,0.0f)，FixedUpdate的执行频率采用系统默认值（0.02s）

动量定理：$f*t=m*v$

- **ForceMode.Force**

  默认方式，使用刚体的质量计算，以每帧间隔时间为单位计算动量，即
  $$
  f*0.02=2*v \\
  v=f*0.01=(0.1,0,0)m/s
  $$

- **ForceMode.Acceleration**

  忽略刚体的实际质量而采用默认值1.0f，即
  $$
  f*0.02=1*v \\
  v=f*0.02=(0.2,0,0)m/s
  $$

- **ForceMode.Impulse**

  此种方式采用瞬间力作用方式，把t的值默认为1，不再采用系统的帧频间隔，即
  $$
  f*1=2*v \\
  v=f/2=(5,0,0)m/s
  $$

- **ForceMode.VelocityChange**

  忽略刚体的实际质量，采用默认质量1.0，同时也忽略系统的实际帧频间隔，采用默认间隔1.0，即
  $$
  f*1=1*v \\
  v=f=(10,0,0)m/s
  $$
