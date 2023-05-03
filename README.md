# 进程管理：电梯调度
> 课程：操作系统
> 
> 老师：张惠娟
> 
> 姓名：蒋钰萱
> 
> 学号：2152216
## 项目介绍
### 项目目的
+ 通过控制电梯调度，实现操作系统调度过程
+ 学习特定环境下多线程编程方法
+ 学习调度算法
### 基本任务
某一层楼20层，有5部互联的电梯。基于线程思想，编写一个电梯调度程序。
### 功能描述
+ 电梯有一些按键：数字键、开门键、关门键、上行键、下行键、报警键
+ 有数码显示器指示当前每部电梯的状态
+ 每层楼（电梯外），有上行、下行按钮
+ 五部电梯相互联结，即当一个电梯按钮按下去时，其他电梯按钮同时点亮，表示也按下去了（在实际实现时，每一层只采用一个上行按钮和一个下行按钮）

具体页面如下图所示：

![image](https://user-images.githubusercontent.com/126655217/235858364-454c8324-ce68-4853-abb2-b478f3a2f4fc.png)

### 开发环境
+ 系统环境：Windows 10
+ 开发软件：PyCharm
+ 开发语言：python 3.9
  + 需要安装的程序包：
    + Python (64bit)
    + tkinter
    + threading
    ```commandline
        import tkinter as tk
        import tkinter.font as tkfont
        from tkinter import ttk
        from tkinter import messagebox
        import time
        import threading
    ```
## 图形界面
本项目使用 `Tkinter` 来开发 GUI 应用程序。
> `Tkinter` 是 Python 的标准图形用户界面（GUI）库，它是 Python 对 Tcl/Tk GUI 工具包的接口。
  
在本项目中，创建了一个名为`Application`的类用来实现图形界面。

在实现图形界面时，主要把界面划分成三个部分，包括电梯内的部分、每层楼电梯外的部分和显示电梯状态的显示器部分。

如电梯内的部分(其他类似，不做展示)：
```commandline
    # 绘制电梯按钮
    def show_eleBotton(self, parent, frm_name):

        def label(frame_parent, text, size=10, bg="white"):
            return tk.Label(frame_parent, text=text, bg=bg, font=tkfont.Font(family="微软雅黑",size=size, weight="bold"))

        def button(frame_parent, text, command, name):
            return tk.Button(frame_parent, text=text, width=30,
                             bg="PowderBlue", fg="white",state=tk.NORMAL,
                             command=command, name=name, font=tkfont.Font(family="微软雅黑", size=9, weight="bold"))
        # 五部电梯的背景框架
        frame = tk.Frame(parent, name=frm_name, width=90, height=580, bg="Tan")
        if frm_name[-1] == "L": # 左侧的按钮1-10
            '''
            click_in 函数有三个参数
            第一个参数是一个字符串，表示按钮所在的位置
            第二个参数是一个整数，表示按钮所在的框架
            第三个参数也是一个整数，表示按钮的编号
            '''

            for i in range(1, 11):
                button(frame, "  %d  " % i, lambda i=i: self.click_in("L", int(frm_name[3]), i),
                       name="%d" % i).pack(padx=8, pady=8)
            button(frame, "  开 ", lambda: self.click_door(int(frm_name[3]), 1),
                   name="open").pack(padx=8, pady=8)
            button(frame, "  报警  ", lambda: self.alarmButton(self),
                   name="alarm").pack(padx=8, pady=8)

            tk.Frame(frame, width=8, bg="Tan").pack(side=tk.LEFT, fill=tk.Y)

            label(frame, frm_name[3] + "号电梯", bg="RosyBrown").pack(anchor=tk.W, expand=True, fill=tk.Y)
        else:   # 右侧的按钮11-20
            for i in range(11, 21):
                button(frame, "  %d  " % i, lambda i=i: self.click_in("R", int(frm_name[3]), i),
                       name="%d" % i).pack(padx=8, pady=8)

            button(frame, "  关  ", lambda: self.click_door(int(frm_name[3]), 0),
                   name="close").pack(padx=8, pady=8)

        # 禁止框架自动调整大小
        # 一定要加上这句代码，不然就乱了
        frame.propagate(False)

        return frame
```
之后，在`if __name__ == "__main__":`中创建一个名为`app`的`Application`类的实例，再进行后续操作。
## 调度算法
### 背景
在进行调度之前，需要创建一个电梯类（`class Elevator`）。

这个类包含用于存储5部电梯各自目标楼层的字典（分为外调度和内调度）。
```commandline
eleTarget = {
    1: {"inside_queue": {}, "outside_up_queue": {}, "outside_down_queue": {}},
    2: {"inside_queue": {}, "outside_up_queue": {}, "outside_down_queue": {}},
    3: {"inside_queue": {}, "outside_up_queue": {}, "outside_down_queue": {}},
    4: {"inside_queue": {}, "outside_up_queue": {}, "outside_down_queue": {}},
    5: {"inside_queue": {}, "outside_up_queue": {}, "outside_down_queue": {}},
}
```
字典中还存储并初始化了相关信息，分别记录了电梯运行的历史状态`self.eleTarget[ele_num]["historyState"]`、电梯是否停止运行`self.eleTarget[ele_num]["stay"]`、电梯门的状态`self.eleTarget[ele_num]["open"]`、电梯的当前所在楼层数`self.eleTarget[ele_num]["nowfloor"]`、电梯运行的方向`self.eleTarget[ele_num]["direction"]`等信息。

在刚开始时，所有电梯的初始状态都在第一层，因此需要初始化每部电梯的当前楼层数为1。
### 设计思路
采用的是LOOK算法，分为内调度和外调度。
内调度是对于电梯内部按钮的调度，外调度是对电梯外每层楼的按钮的调度。

#### LOOK算法思想
LOOK算法是操作系统中磁盘调度的一种算法。LOOK算法用于磁盘调度中时，是指读/写头在开始由磁盘的一端向另一端移动时，随时处理所到达的任何磁道上的服务请求，直到移到最远的一个请求的磁道上。一旦在前进的方向上没有请求到达，磁头就反向移动。在磁盘另一端上，磁头的方向反转，继续完成各磁道上的服务请求。这样，磁头总是连续不断地在磁盘的两端之间移动。

LOOK算法用于电梯调度中，电梯其实就是LOOK算法中的磁头，楼层上的请求就是磁道上的服务请求。这就是磁盘调度LOOK算法用于电梯调度的思想

#### 内调度
简单描述就是，电梯一次朝一个方向运行，待这个方向没有任何请求时，再查看另一方向是否有请求。如果另一个方向有请求，电梯调头；如果另一个方向也没有请求，电梯停止。

通过字典记录电梯上一次运行的历史状态`eleTarget[ele_num]["historyState"]`，如果历史状态为"up"，表示电梯在上升状态，如果在电梯对应的当前楼层及以上楼层有请求，则电梯上升。同理可知电梯下降的情形。
这是对应电梯当时没有运行状态的情况。

如果电梯正在上升，检查上升队列是否为空，如果上升队列为空，停止运行；如果不为空，继续上升。下降的情况同理。

以下函数的作用就是检查请求，返回任务进行方向。
```commandline
# 检查请求队列是否为空，不为空则返回任务方向
def askForEmpty(ele_num, cur_floor):

    flag = "empty"

    if ele.eleTarget[ele_num]["historyState"] == "down" or ele.eleTarget[ele_num]["historyState"] == "stop":
        for i in range(1, cur_floor):
            if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                flag = "down"
                break
            if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                flag = "down"
                break
            if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                flag = "down"
                break
        if flag=="empty":
            for i in range(cur_floor, 20):
                if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                    flag = "up"
                    break
                if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                    flag = "up"
                    break
                if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                    flag = "up"
                    break

    if ele.eleTarget[ele_num]["historyState"] == "up":
        for i in range(cur_floor , 20):
            if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                flag = "up"
                break
            if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                flag = "up"
                break
            if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                flag = "up"
                break
        if flag=="empty":
            for i in range(1, cur_floor):
                if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                    flag = "down"
                    break
                if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                    flag = "down"
                    break
                if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                    flag = "down"
                    break


    # 针对按下当前楼层按钮的情况
    if ele.eleTarget[ele_num]["inside_queue"][cur_floor] == 1:
        flag = "this"
        ele.eleTarget[ele_num]["inside_queue"][cur_floor] = 0
        refresh_InButton(ele_num, cur_floor)
    if ele.eleTarget[ele_num]["outside_up_queue"][cur_floor] == 1:
        flag = "this"
        ele.eleTarget[ele_num]["outside_up_queue"][cur_floor] = 0
    if ele.eleTarget[ele_num]["outside_down_queue"][cur_floor] == 1:
        ele.eleTarget[ele_num]["outside_down_queue"][cur_floor] = 0
        flag = "this"

    return flag
```
以下为调用`askForEmpty`函数的内容：
```commandline
# 电梯停止
if ele.eleTarget[ele_num]["direction"] == Direction.STOP:
    flag_this_floor = False
    # 到达当前楼层，开门
    if askForEmpty(ele_num, ele.eleTarget[ele_num]["nowfloor"]) == "this":
        ele.eleTarget[ele_num]["state"] = State.STOP
        ele.eleTarget[ele_num]["direction"] = Direction.STOP
        ele.eleTarget[ele_num]["open"] = True
        flag_this_floor = True
    # 指令为空，停止运行
    if askForEmpty(ele_num, ele.eleTarget[ele_num]["nowfloor"]) == "empty":
        ele.eleTarget[ele_num]["state"] = State.STOP
        ele.eleTarget[ele_num]["direction"] = Direction.STOP
    # 电梯指令为上升，向上运行
    if askForEmpty(ele_num, ele.eleTarget[ele_num]["nowfloor"]) == "up":
        ele.eleTarget[ele_num]["state"] = State.RUNNING
        ele.eleTarget[ele_num]["direction"] = Direction.UP
        ele.eleTarget[ele_num]["historyState"] = "up"
    # 指令为下降，向下运行
    if askForEmpty(ele_num, ele.eleTarget[ele_num]["nowfloor"]) == "down":
        ele.eleTarget[ele_num]["state"] = State.RUNNING
        ele.eleTarget[ele_num]["direction"] = Direction.DOWN
        ele.eleTarget[ele_num]["historyState"] = "down"
    # 如果电梯门是打开的，且这层楼无任务，关闭电梯门
    if ele.eleTarget[ele_num]["open"] and flag_this_floor == False:
        ele.eleTarget[ele_num]["open"] = False
```
对于电梯内调度的请求指令，有以下函数：
```commandline
# 刷新内部指令
def refresh_in():
    for ask in inter_instruction:
        ele.eleTarget[ask["ele_num"]]["inside_queue"][ask["target_floor"]] = 1
        ele.eleTarget[ask["ele_num"]]["asked"] = True
    
    if ele.eleTarget[ask["ele_num"]]["state"] == State.STOP and ele.eleTarget[ask["ele_num"]][
    "direction"] == Direction.STOP:
        # 如果电梯当前楼层小于目标楼层
        if ele.eleTarget[ask["ele_num"]]["nowfloor"] < ask["target_floor"]:
            # 将电梯的状态修改为运行状态，并将方向设置为上行
            ele.eleTarget[ask["ele_num"]]["state"] = State.RUNNING
            ele.eleTarget[ask["ele_num"]]["direction"] = Direction.UP
        # 如果电梯当前楼层大于目标楼层
        elif ele.eleTarget[ask["ele_num"]]["nowfloor"] > ask["target_floor"]:
            # 将电梯的状态修改为运行状态，并将方向设置为下行
            ele.eleTarget[ask["ele_num"]]["state"] = State.RUNNING
            ele.eleTarget[ask["ele_num"]]["direction"] = Direction.DOWN
    # 清空 inter_instruction 列表
    inter_instruction.clear()
```
当电梯处于上升的运行状态时（下降同理），对于内调度有：
```commandline
# 如果电梯处于运行状态
elif ele.eleTarget[ele_num]["state"] == State.RUNNING:
    # 将要继续向上移动
    if ele.eleTarget[ele_num]["direction"] == Direction.UP:
        ele.eleTarget[ele_num]["historyState"] = "up"
        # 检查当前楼层是否有人要上电梯
        if ele.eleTarget[ele_num]["inside_queue"][ele.eleTarget[ele_num]["nowfloor"]] == 1:
            # 如果前方无任务或者电梯到达顶层
            if askForFree(ele_num, ele.eleTarget[ele_num]["nowfloor"]) or ele.eleTarget[ele_num]["nowfloor"] == 20:
                ele.eleTarget[ele_num]["state"] = State.STOP
                ele.eleTarget[ele_num]["direction"] = Direction.STOP
                refresh_InButton(ele_num, ele.eleTarget[ele_num]["nowfloor"])
            # 如果前方有任务，电梯会继续向上移动
            elif not askForFree(ele_num, ele.eleTarget[ele_num]["nowfloor"]):
            # else:
                ele.eleTarget[ele_num]["state"] = State.STOP
                ele.eleTarget[ele_num]["direction"] = Direction.UP
                refresh_InButton(ele_num, ele.eleTarget[ele_num]["nowfloor"])
            # 电梯到达某一楼层时，将该楼层的内部指令队列中的值设为 0，表示该楼层的任务已经完成
            ele.eleTarget[ele_num]["inside_queue"][ele.eleTarget[ele_num]["nowfloor"]] = 0
            ele.eleTarget[ele_num]["open"] = True
            
            #......            
```

#### 外调度
创建一个名为`refresh_out`的函数，它用于处理外部请求。函数中首先创建了一个字典，用于存储可能满足请求的电梯及其与请求楼层之间的距离。然后，对于每个外部请求，检查其目标方向是否为上行。如果是，则遍历所有电梯，检查当前电梯是否处于停止状态或上行状态且当前楼层小于等于请求楼层。如果满足条件，则调用`distance`函数来计算该电梯与请求楼层之间的距离，并将结果存储在字典中。最后，检查字典是否为空，如果不为空，则对其进行排序，以找到距离最近的电梯，并更新该电梯的`outside_up_queue`，并从外部请求指令`exter_instruction`列表中删除该请求。

如果目标方向为下行，则执行类似的操作，只是将上行状态改为下行状态，将小于等于改为大于等于。

部分代码如下：
```commandline
# 用于存储满足请求的电梯及其与请求楼层之间的距离
eleTemp = {}
for ask in exter_instruction:
    # 检查请求的目标方向是否为上行
    if ask["target_direction"] == "U":
        for ele_num in range(1, 6):
            # 检查当前电梯是否处于停止状态
            if ele.eleTarget[ele_num]["direction"] == Direction.STOP:
                # 调用 distance 函数来计算该电梯与请求楼层之间的距离，并将结果存储在 eleTemp 字典中
                eleTemp[ele_num] = distance(ele_num, ask["current_floor"])
            # 检查当前电梯是否处于上行状态且当前楼层小于等于请求楼层
            if ele.eleTarget[ele_num]["direction"] == Direction.UP and ele.eleTarget[ele_num]["nowfloor"] <= int(ask[
                "current_floor"]):
                # 调用 distance 函数来计算该电梯与请求楼层之间的距离，并将结果存储在 eleTemp 字典中
                eleTemp[ele_num] = distance(ele_num, ask["current_floor"])

#......

# 进行排序，以找到距离最近的电梯
find = sorted(eleTemp.items(), key=lambda item: item[1])
# 更新该电梯的 outside_up_queue，并从 exter_instruction 列表中删除该请求
ele.eleTarget[find[0][0]]["outside_up_queue"][int(ask["current_floor"])] = 1
exter_instruction.remove(ask)
```
## 响应
主要包括对按钮的响应和对电梯调度的响应。
### 外调度的按钮响应
(内调度的按钮响应也类似)
```commandline
# 按钮响应--外调度
def click_out(self, target_direction):
    # 获取了名为"frm_mainbody"的子组件的引用，并将其存储在变量frm中
    # 可以通过变量frm来访问和操作这个组件
    frm = self.root.children["frm_mainbody"]
    # 获取了frm组件的名为"frm_outside"的子组件的引用，并将其存储在变量frm_out中
    frm_out = frm.children["frm_outside"]
    out = frm_out.children["outside"]
    box = out.children["floor_select_box"]
    if target_direction == "U":
        button = frm_out.children["ele_up"]
    else:
        button = frm_out.children["ele_down"]
    # 如果按钮状态正常
    if button["state"] == tk.NORMAL:
        # 禁用按钮
        button["state"] = tk.DISABLED
        # 设置背景色
        button["bg"] = "Teal"

    # 获取当前楼层的值
    current_floor = box.get()
    # 创建一个字典，其中包含目标方向和当前楼层两个键值对
    ask = {"target_direction": target_direction, "current_floor": current_floor}
    # 添加到exter_instruction列表中
    exter_instruction.append(ask)
```
### 电梯内部开/关门按钮、报警按钮的响应
调用`click_door`函数来处理电梯内部开门/关门按钮的单击事件。

调用`alarmButton`函数处理报警按钮的单击事件。
```commandline
button(frame, "  开 ", lambda: self.click_door(int(frm_name[3]), 1),name="open").pack(padx=8, pady=8)
button(frame, "  报警  ", lambda: self.alarmButton(self),name="alarm").pack(padx=8, pady=8)
button(frame, "  关  ", lambda: self.click_door(int(frm_name[3]), 0),name="close").pack(padx=8, pady=8)                   
```
```commandline
def alarmButton(self):
    self.show_info("已报警！")
def show_info(message=""):
    messagebox.showinfo("紧急情况", message)

def click_door(ele_num=0, open_=0):
    flag = 0
    if ele.eleTarget[ele_num]["state"] == State.STOP and ele.eleTarget[ele_num]["direction"] == Direction.STOP:
        flag = 1
    if ele.eleTarget[ele_num]["state"] == State.STOP and ele.eleTarget[ele_num]["direction"] == Direction.UP:
        flag = 1
        ele.eleTarget[ele_num]["stay"] = True
    if ele.eleTarget[ele_num]["state"] == State.STOP and ele.eleTarget[ele_num]["direction"] == Direction.DOWN:
        flag = 1
        ele.eleTarget[ele_num]["stay"] = True
    if flag == 1:
        if open_ == 0:
            ele.eleTarget[ele_num]["open"] = False
        if open_ == 1:
            ele.eleTarget[ele_num]["open"] = True
```
### 电梯调度的响应
+ 如果电梯正在运行，不能开门。
+ 如果电梯当前处于暂停状态：
  + 将要上升，判断前方是否有任务且电梯门是否关闭，如果满足条件，修改电梯的状态为运行状态，修改电梯运行方向为向上，否则，电梯状态为停止。
  + 电梯将要下降时同理。
  + 当电梯将要停止时，分为5种情况：
    + 到达目标楼层，开门。
    + 无请求指令，电梯停止运行。
    + 有上升的电梯指令，修改电梯状态为运行，方向为上升。
    + 有下降的电梯指令，修改电梯状态为运行，方向为下降。
    + 如果电梯门是开着的，且当前楼层无任务，关门。
+ 如果电梯当前处于运行状态：
  + 将要上升：
    + 检查当前楼层是否有人要下电梯（电梯内请求指令）。如果有人，且前方有任务，则电梯继续上升。如果有人，但前方无任务或电梯已经到达顶层，电梯停止运行。
    + 检查当前及以上楼层是否有人要上电梯（电梯外每层楼的请求指令）。再检查当前楼层以下的楼层是否有人上电梯。设置不同的电梯运行方向。
  + 将要下降（同理）

随后，根据电梯的运行方向，加或减相应楼层。
```commandline
for ele_num in range(1, 6):
    # 直接移动电梯
    if ele.eleTarget[ele_num]["state"] == State.RUNNING:
        if ele.eleTarget[ele_num]["direction"] == Direction.UP:
            ele.eleTarget[ele_num]["historyState"] = "up"
            ele.eleTarget[ele_num]["nowfloor"] += 1
        elif ele.eleTarget[ele_num]["direction"] == Direction.DOWN:
            ele.eleTarget[ele_num]["historyState"] = "down"
            ele.eleTarget[ele_num]["nowfloor"] -= 1
```
控制电梯移动的函数`move()`，及其他检查电梯状态的函数均位于`control`函数中。
## 多线程
程序创建了两个线程，一个是名为`control`的线程，一个是名为`all_refresh`（用来刷新按钮状态、显示屏、内外调度的请求指令）的线程。

对于名为`control`的函数，在其内部定义一个无限循环的函数`funOfControl`，频繁地获取互斥锁，调用`move`函数，最后释放互斥锁。然后声明一个名为`control`的线程，将其目标函数设置为`funOfControl`，并将这个线程的`daemon`属性设为`True`，表示这个线程是一个守护线程。最后，调用这个线程的`start`方法来启动这个线程。
> **互斥锁**
> 是一种同步原语，用来保护对共享资源的访问。当多个线程需要访问共享资源时，它们可以使用互斥锁来确保在同一时间只有一个线程能够访问共享资源。这样，就可以避免出现竞态条件和数据不一致的问题。因此可以确保在同一时间只有一个线程能够调用`move`函数。
> 
> **守护线程**
> 是一种特殊的线程，它会在主程序退出时自动终止。这意味着，当主程序退出时，不需要等待守护线程执行完毕，它会自动终止。

> **threading.Lock**是直接通过`_thread`模块扩展实现的。
> 当锁在被锁定时，它并不属于某一个特定的线程。
> 
> 锁只有“锁定”和“非锁定”两种状态，当锁被创建时，是处于“非锁定”状态的。当锁已经被锁定时，再次调用`acquire()`方法会被阻塞执行，直到锁被调用`release()`方法释放掉锁并将其状态改为“非锁定”。
> 同一个线程获取锁后，如果在释放锁之前再次获取锁会导致当前线程阻塞，除非有另外的线程来释放锁，如果只有一个线程，并且发生了这种情况，会导致这个线程一直阻塞下去，即形成了死锁。所以在获取锁时需要保证锁已经被释放掉了，或者使用递归锁来解决这种情况。
> 
> **acquire(blocking=True, timeout=-1)**：获取锁，并将锁的状态改为“锁定”，成功返回`True`，失败返回`False`。当一个线程获得锁时，会阻塞其他尝试获取锁的线程，直到这个锁被释放掉。`timeout`默认值为-1，即将无限阻塞等待直到获得锁，如果设为其他的值时（单位为秒的浮点数），将最多阻塞等待`timeout`指定的秒数。当`blocking`为`False`时，`timeout`参数被忽略，即没有获得锁也不进行阻塞。
> 
> **release()**：释放一个锁，并将其状态改为“非锁定”，需要注意的是任何线程都可以释放锁，不只是获得锁的线程（因为锁不属于特定的线程）。`release()`方法只能在锁处于“锁定”状态时调用，如果在“非锁定”状态时调用则会报`RuntimeError`错误。

```commandline
# 1秒执行一次
def funOfControl():
    while True:
        time.sleep(1)
        # 获取互斥锁
        mutex.acquire()
        move()
        # 释放互斥锁
        mutex.release()

t = threading.Thread(target=funOfControl, name="control")
t.daemon = True
t.start()
```
对于名为`all_refresh`的函数，有：
```commandline
# 每0.1秒执行一次
def funOfRefresh():
    while True:
        time.sleep(0.1)
        someRefresh()   # 刷新指令队列以及电梯状态---调度指令
        refreshButton() # 根据当前下拉选框选择的外楼层号刷新上下按钮状态
        refreshScreenDetail()   # 根据各个楼层状态刷新显示屏

t = threading.Thread(target=funOfRefresh, name="all_refresh")
t.daemon = True
t.start()
```
在执行过程中（`if __name__ == "__main__":`的里面），创建了一个互斥锁，用于允许多个线程同步访问共享资源。然后，使用`app.root.after`方法来启动两个线程：`control`和`all_refresh`。这两个线程将在主循环启动后立即开始运行。最后，调用`app.root.mainloop`方法来启动`Tkinter`的主循环，用于启动GUI并处理用户交互。
```commandline
# 启动互斥锁，允许多个线程同步访问共享资源
mutex = threading.Lock()
# 启动2个线程--control、all_refresh
# 将在主循环启动后立即开始运行
app.root.after(0, control)
app.root.after(0, all_refresh)
# 启动Tkinter的主循环--启动GUI并处理用户交互
app.root.mainloop()
```
## 项目进行过程中遇到的问题及解决方法
+ **内调度**
  + 一开始的时候没有实现让电梯在某一方向运行完了之后再转另一方向，出现了饥饿现象。
  + 解决办法是调整了`askForEmpty`函数
      + 一开始的`askForEmpty`函数如下所示：**（会出现当电梯向上运行时，不管上面有多少层有请求指令，一旦遇到向下的指令，就会立即向下，导致向上的指令不能及时调度的情况）**
      ```
      flag = "empty"
  
      for i in range(cur_floor , 20):
          if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
              flag = "up"
              break
          if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
              flag = "up"
              break
          if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
              flag = "up"
              break
    
      for i in range(1, cur_floor):
          if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
              flag = "down"
              break
          if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
              flag = "down"
              break
          if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
              flag = "down"
              break
      ```
      + 调整思路为，当电梯之前处在上升/下降状态时，确定上升/下降的请求为空之后，再去处理下降/上升的请求指令。调整后的代码如下所示（已经成功解决问题）。
      ```
      flag = "empty"

      if ele.eleTarget[ele_num]["historyState"] == "down" or ele.eleTarget[ele_num]["historyState"] == "stop":
          for i in range(1, cur_floor):
              if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                  flag = "down"
                  break
              if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                  flag = "down"
                  break
              if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                  flag = "down"
                  break
          if flag=="empty":
              for i in range(cur_floor, 20):
                  if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                      flag = "up"
                      break
                  if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                      flag = "up"
                      break
                  if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                      flag = "up"
                      break

      if ele.eleTarget[ele_num]["historyState"] == "up":
          for i in range(cur_floor , 20):
              if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                  flag = "up"
                  break
              if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                  flag = "up"
                  break
              if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                  flag = "up"
                  break
          if flag=="empty":
              for i in range(1, cur_floor):
                  if ele.eleTarget[ele_num]["inside_queue"][i] == 1:
                      flag = "down"
                      break
                  if ele.eleTarget[ele_num]["outside_up_queue"][i] == 1:
                      flag = "down"
                      break
                  if ele.eleTarget[ele_num]["outside_down_queue"][i] == 1:
                      flag = "down"
                      break
      ```
## 运行方法
在确认安装此前提到的python包后，运行`elevatorDispatch.py`文件，会出现如下所示图形界面。

![image](https://user-images.githubusercontent.com/126655217/235858305-414f22e9-4d99-4cac-8139-c197dd972f10.png)

图形页面中间是5部电梯及对应电梯内的按钮，下面是5部电梯对应的显示屏，右边是电梯外每层楼的按钮，根据需要按相应的按钮即可。
