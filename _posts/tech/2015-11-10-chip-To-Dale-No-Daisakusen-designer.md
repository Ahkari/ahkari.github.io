---
layout: post
title: <松鼠大战>标准FC游戏设计
category: 技术
tags: 游戏设计 javascript 
description: 基于gamemei游戏平台做个游戏，还是我最喜欢的游戏。
---

### 前言

本来没有这段的，我只是想吼一声，这是我小时候最喜欢的游戏！
想当年偷偷摸摸去小伙伴家玩一下都能回味半天，没错，就是这么惨！
![松鼠大战](http://7xny7k.com1.z0.glb.clouddn.com/s1.png)

### 功能预估
基于一个第三方的平台gamemei（也是强行把我司说成是第三方），我们可以快速做一个游戏。
我们不用关注大部分2D游戏引擎所做的事，专注于游戏的实现即可。
在做这个游戏之前我特意又下了个NES模拟器玩了下，回想下游戏的操作方式。
最终我选择了第一关的boss战作为我要还原的场景。
![第一关boss](http://7xny7k.com1.z0.glb.clouddn.com/s4.png)
这场面是不是很熟悉？
我新制定了一个游戏规则：
1. player是帽子松鼠，具有原游戏的一切状态，上下左右，搬箱子扔箱子以及跳跃。还可以躲进箱子里。
![帽子松鼠](http://7xny7k.com1.z0.glb.clouddn.com/s2.png)
2. 物理规则遵循原游戏，平台具有单向性，即只能从下至上跳上而不能从上往下落下（下蹲+跳跃的落下方式暂不支持，其实是我忘了搞了，以后有机会再弄）。例如下图,松鼠可以从锅跳跃到餐架上，却不会掉下来。
![单向跳跃](http://7xny7k.com1.z0.glb.clouddn.com/s3.png)
3. 箱子可以搬，同时支持左右上三个方向扔出去。
4. boss是花衬衫松鼠，绑着氢气球不断朝player撞去。player被撞到即掉血，如果boss被player扔出去的箱子砸到就会掉血，血量归零另一方胜利。双方掉血之后都有一秒左右的无敌状态。

虽然一开始不是这么设计的，但是到最后这个最可行，可玩性也不错。

那就是他了。

### 重点模块实现

1. 平台单向碰撞
2D物理引擎为了真实，不存在什么矩形固体平台单向通过的特性。只能靠我们控制平台的单通性。
这里我是这样处理的：
![实现单向碰撞](http://7xny7k.com1.z0.glb.clouddn.com/s5.png)
在每个平台稍高点的地方都会有传感线，分别控制下面和自己挨得最近的平台。
我们松鼠如果要跳上平台，那自身跳跃高度一定要足够，也就是自己要跳到比平台高一头的位置。
我们平时将除了脚下踩着的平台全部隐藏，只有跳跃后撞到到比某平台高一头的传感线时，其控制的平台才会显示出来。于是就出现了单通性的“平台”。
我们每次起跳都会将所有平台再次隐藏，周而复始，每个平台其是否实际存在都是动态的。
需要注意的是，为了尽可能贴近游戏的感觉，我把平台设定的非常非常细（1像素）。但是当他显现出来的却是完整的固体特性，也就是说你撞上他一像素宽的侧面也会被强制速度归零，这个在后续设置松鼠状态时要作为一个特例处理。

2. 松鼠状态管理
player帽子松鼠的状态是游戏最重要的部分。
因为FC游戏的控制方式一直以来都是 上下左右 和 BA ，所以思考很久后，还是觉得对于所有的输入都做状态管理比较好。
对于上下左右（之后就用WASD代替），和BA（之后用JK代替）每个的按下和抬起事件触发时，
我们执行如下操作：
记录下按键状态 —— 进入对应按键处理程式 —— 获取当前状态 —— 计算按键执行之后的状态 —— 交付给状态机统一处理。
那么状态机是干啥的？
其实就干了简单的三件小事：
执行速度变化，更改松鼠状态，根据改变后的状态切换显示图像。
简而言之，图像依附于状态，状态变更必然会导致图片显示的变化，哪怕是同一张图。
而物理速度并不是状态值，因为实际中会有撞墙等意外因素，不能说撞了墙水平速度为0了咱们就得显示个撞墙状态对不对，撞墙时我们应当继续显示小松鼠在正常的朝着墙努力地走。

        //状态机, 根据输入改变状态
        game.vars_.stateMachine = function stateMachine( setObj , speed , nextState , timeDelay ){
            var self = setObj ;
            console.log('状态改变为') ;
            console.log( nextState ) ;
            self.setSpeed( speed.x , speed.y ) ;
            self.setVar( 'boxing' , nextState['boxing'] ) ;
            self.setVar( 'state' , nextState['state'] ) ;
            self.setVar( 'direction' , nextState['direction'] ) ;
            game.vars_.changeDisplay( setObj , nextState , timeDelay ) ;
        }

    我们看下我们给松鼠指定的三个状态值。这三个属性值三三随机组合能囊括松鼠所有的状态。
    状态属性一： boxing 即是否搬着箱子。0 没有； 1 有。
    状态属性二： state 即单体动作。stand站着；walk行走；jump跳跃；lying下蹲；jumpNoSpeed原地跳跃（这个展示的图片和jump一样，但确实是一个需要独立出来的状态，类比stand之于walk）；
    状态属性三： direction 即朝向。0 朝左； 1 朝右。

    假设我们搬着箱子在向左跳跃的过程中按了下D键，
    我们获取当前状态，不出所料是这样：

    {
        boxing : 1 ,
        state : 'jump' , 
        direction : 0 
    }

    我们根据当前按键状态，以及其他因素，判断之后应该变成这样的状态：
    
    {
        boxing : 1 ,
        state : 'jump' ,
        direction : 1
    }

    并且应当给个这样的速度
    
    {
        x : 140 , //向右140px/s
        y : 0 + self.speed.y  //y方向速度保持不变
    }

    最后交付给状态机，状态机帮我们设了速度，改了状态，交付给`game.vars_.changeDisplay()`公共方法来改变显示的图形。
    我们根据更改后的状态傻瓜式if语句查询，最后找到要更改的图形`boxJumpRight_default`：

    ...
    }else if( nextState['boxing'] === 1 ){
        switch ( nextState['state'] ){
        ...
        case 'jump':
            if ( nextState['direction'] === 1 ){
                self.changeSprite('boxJumpRight_default');
            }
        ...

    我的上帝我发誓我找不出比这更蠢的代码了，而且我还想不到特别有效的优化办法。
    最后我们看下图片，就知道我们松鼠有多少对应状态了233。
    ![松鼠状态](http://7xny7k.com1.z0.glb.clouddn.com/s7.png)

3. 盒子状态管理
管理完松鼠之后，盒子实在是太简单了。
因为其无复杂的状态，只能在几个状态间切换，甚至用一个变量就可以表示全部。
0：正常随着流水运动，不能给举起
1：正常随着流水运动，但和松鼠碰撞了，能给举起
2：给举起的状态
3：扔出去的状态，此状态的盒子砸到boss会掉血。

4. 动画后摇实现
我们观察到，松鼠落地会有个趴下几百毫秒的动画过程。
扔出箱子的时候也会有半秒左右的时间执行扔箱子动画。
这是动作后摇。
为什么不是状态的一种呢，因为是个过渡动作，执行完之后会回归原动作或跳至下个动作。
我们没有任何一个输入的信息会导致两次状态变化。
所以这个只能作为一个不改变状态，单纯改变角色展示的后摇动作。
我们设计的动画后摇需要有如下特性：
    * 我们让其执行就执行
    * 一般给一个固定的执行时长，在这个时长内可以改变状态与显示
    * 执行时长完的时候，判断当前动画如果还是后摇动画，说明执行期间内没有图片的改变，我们只需回复图片为执行前的图片即可。

            //强制执行指定后摇动画，参数一是要执行的后摇动画名，参数二是API提供的对象名。
            game.vars_.execute = function execute(anime,self){
                if ( anime === 'throwRight'){
                    var storeSprite  ;
                    setTimeout(function(){
                        storeSprite = self.spriteName ;//确保正确存储了后摇前的展示图片名
                        self.changeSprite('throwRight_default');
                    },0) ;
                    setTimeout(function(){
                        if ( self.spriteName === 'throwRight_default' ){
                            self.changeSprite( storeSprite ) ;//将后摇前的图片展示回去
                        }
                    },300) ;//这个向右扔箱子的持续时间为300毫秒
                }
            }
    
5. 血量控制与免疫状态的实现
血量我们统一用一个方法控制下：
在每次我们让某个角色掉血的时候调用这个方法，指定掉血的角色，我们就会根据这个角色当前的血量（掉血后）来改变其血量显示。
这时我们将其“免疫”属性设为开启，设定1秒后解除“免疫”。当然，在这个方法开始我们就会对免疫属性的是否开启来判断是否要掉血。
这个功能也没什么可说的。

至此我们的核心方法都已实现。
最复杂的是松鼠的状态管理，针对每个状态下不同的按键都有一次状态计算，所以后台有大约1500行的js代码，大部分都是状态参数，这部分的正确性直接关联到角色表现正确与否。
如果角色表现异常，根据状态记录很快就能找到是在哪一步出的错。
![状态记录](http://7xny7k.com1.z0.glb.clouddn.com/s6.png)


### 基于第三方平台的其他细节的实现

1. 物理环境
Box2D提供，位于一个Y方向30N的重力环境下，不过讽刺的是游戏里面只有player一人承受了重力。。。

2. 事件机制
gmx引擎提供，本游戏只支持了WASDJK的键盘按下与抬起事件

3. 碰撞检测
gmx引擎提供，约60ms全局对绑定了碰撞检测的物体检测一次。本游戏中碰撞检测过多，有判别不出来的情况。

4. 序列帧动画
gamemei前端提供，0.1s每帧，按照给定的图片顺序显示。游戏里的动起来的图片都是序列帧动画。

5. 消息机制
webSocket实现，此游戏中盒子调到场景之外的时候碰撞到指定物体的时候就会触发消息发送。
盒子接受到消息后重启运动路径。

6. 场景切换
gmx实现，可以在指定场景之间跳转。


### 最后

这游戏我把boss设定成160px/s的移速，
这种高速下我玩起来还觉得boss战五渣，你们这就玩不过去啦？
你就玩玩**消消乐** **flappybird**咯。逃~