## 1. 基本结论

在 micro-USB OTG 中，Host / Device 的角色判断主要依赖 ID 脚：

```
ID = GND   → 本端作为 Host / A-device
ID 悬空    → 本端作为 Device / B-device
```

也就是说：

- micro-A plug：ID 接地，本端进入 Host 模式
    
- micro-B plug：ID 悬空，本端进入 Device 模式
    

---

## 2. micro-USB 接口的 5 个引脚

micro-USB 接口一共 5 个引脚：

|   |   |   |
|---|---|---|
|引脚|名称|说明|
|1|VBUS|5V 电源|
|2|D-|USB 数据线负|
|3|D+|USB 数据线正|
|4|ID|OTG 角色识别脚|
|5|GND|地|

其中：

- VBUS：供电线
    
- D+ / D-：数据通信线
    
- GND：地线
    
- ID：用于 OTG 场景下识别本端角色
    

---

## 3. micro-A / micro-B 插头与 ID 脚关系

### micro-A 插头

micro-A 插头内部将 ID 脚接地：

```
ID → GND
```

因此插入后，设备检测到 ID 为低电平，本端作为：

```
Host
```

---

### micro-B 插头

micro-B 插头内部 不连接 ID 脚，因此 ID 处于悬空状态：

```
ID → Float
```

因此插入后，设备检测到 ID 悬空，本端作为：

```
Device
```

---

## 4. micro-AB 插座的作用

micro-AB 插座可以同时接受：

- micro-A 插头
    
- micro-B 插头
    

所以使用 micro-AB 插座的设备，理论上可以根据插入的线缆类型切换角色。

对应关系如下：

|   |   |   |
|---|---|---|
|插头类型|ID 状态|本端角色|
|micro-A plug|接地|Host|
|micro-B plug|悬空|Device|

---

## 5. 常见误区

### 误区 1：micro-A 和 micro-B 只是外形不同

不对。 两者关键区别在于 ID 脚连接方式不同。

### 误区 2：ID 脚电平是 Host 提供的

不对。 ID 状态是由 插头结构决定 的，不是对端主动输出的。

### 误区 3：ID 悬空表示对端一定是 Host

不够准确。 更准确地说：

```
ID 悬空 → 本端作为 Device
```