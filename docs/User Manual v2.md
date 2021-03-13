![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002141256140.png)

@[TOC]

教程非常简单，简单的实现仅需几十行代码。如果有什么不理解的地方，毫不犹豫直接提 issue。 
在开始之前，我们需要了解一下GoldsrcPhysics是什么。GoldsrcPhysics 简称 gsphysics，是一个为**基于 Goldsrc 引擎构建的游戏**提供物理效果的**客户端**物理系统，HLMod 的开发者可以在 HLSDK 客户端模块仅使用几个API就可以获得由GoldsrcPhysics提供的布娃娃效果。【当前版本仅实现了布娃娃】
   
这时候有人要问了，CS1.6也是 Half-Life1 的一个 Mod ，那么CS1.6可以拥有布娃娃吗？这是有可能的，虽然我们没有 CS 的源码，但是使用 [metahook](https://tieba.baidu.com/f?kw=metahook&ie=utf-8) 是可以对 CS 动手脚的。通过 hook 一些函数改成自己的实现，调用这套系统提供的 API 即可。
   > 注意：此教程不会教您在具体哪一行调用某个API，需要您具有一定的 HLMod 的开发经验，您自己根据您的游戏逻辑在合适的地方调用本系统的API为您提供服务。
# 这个系统是如何工作的

# 在您的 HLMod 使用

## 一、安装模块
1. [下载](https://github.com/anchurcn/GoldsrcPhysics/releases)并解压 gsphysics。里面含有SDK以及依赖库。
2. 将 gsphysics文件夹放到半条命的根目录下，和 hl.exe 同级。
3. 确保本机已经安装有.NET 4.6.1运行时，较新版的 Win10 自带，如果没有则去[官网下载](https://dotnet.microsoft.com/download/dotnet-framework/net461)。[.NET Framework 系统要求](https://docs.microsoft.com/zh-cn/dotnet/framework/get-started/system-requirements)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021031015063845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI3NzkzODU=,size_16,color_FFFFFF,t_70)
## 二、编写代码
 1. 打开您的客户端项目
 1. 将 gsphysics/sdk 目录下的两个文件 physics.h;physics.cpp 添加进您的项目
 1. 基本API参考：

下面列出的API已经经过测试，未列出的API自己尝试。

GoldsrccPhysics 的所有API都声明在 physics.h 文件里，全局变量 gPhysics 含有所有的API，通过`gPhysics.XXXXXXX();`调用。
> 注意！ 在调用 gPhysics 里的函数之前，先调用`InitPhysicsInterface(NULL);`初始化 gPhysics 。这个函数将会使用函数指针填充 gPhysics，参数 msg 为 NULL 即可暂未有实际作用。
* 初始化系统
`函数原型：void(_stdcall* InitSystem)(const char* modFolder, void* pEngineStudioAPI);`
> 在游戏载入之后调用，初始化物理系统。在 Mod dll 的生命周期内仅需调用一次。
> modFolder 为你的mod 文件夹名称字符串
> pEngineStudioAPI 为指向 engine_studio_api_t 的指针
> 以上信息都能在HL sdk 中的StudioModelRenderer.h/和StudioModelRenderer.cpp中找到。
> 例：`gPhysics.InitSystem("valve", &IEngineStudio);`
> 客户端 dll 被载入之后会调用一次 HUD_Init 和 StudioRenderer::Init 。本人测试时是在StudioRenderer::Init 中调用。

* 为物理世界载入地图的碰撞体
`void(_stdcall* ChangeLevel)(char* mapName);`
> 每次载入地图都需要调用这个函数为物理世界加载当前地图的地形碰撞体。调用此函数会清除上一个地图的碰撞体（如果有）。
> mapName 给定地图名（没有后缀名以及路径）
> 可在HUD_VidInit()函数中调用。引擎会在加载地图的时候调用一次HUD_VidInit。但是调整分辨率的时候也会调用，注意检查mapName 不为null。

*  为玩家实体创建布娃娃控制器
`void(_stdcall* CreateRagdollControllerModel)(int entityId, void* model);`
> entityId 实体ID
> model 实体所用模型的指针
> 调用此函数会在物理系统内部给这个实体创建给定模型的布娃娃控制器。后续的StartRagdoll等函数都是在这个控制器上操作，如果没有为实体创建布娃娃控制器则调用StartRagdoll没有任何作用。
>其实 除了玩家，只要是使用人类模型的实体都可以，如npc中的barney，scientist，zombie等。如果这个实体将会有布娃娃的状态（或者说这个实体将会死亡），调用这个函数为这个实体创建布娃娃控制器。
>注意：实体更换模型的时候也要使用此函数更换布娃娃控制器的模型绑定。

*  为实体激活布娃娃效果
`void(_stdcall* StartRagdoll)(int entityId);`
> 当实体死亡，调用此函数为已创建布娃娃控制器的实体激活布娃娃效果。接下来这个实体的模型将不受动画系统的控制，而是使用物理引擎计算模型的动作。
> 可以在实体生命值为0的时候调用、播放了死亡动画的第一帧的时候调用或者触发了某些死亡事件的时候调用，取决于你的游戏实现。

* 设置布娃娃的第一帧动作
`void(_stdcall*SetPose)(int entityId, float* pBoneWorldTransform);`
> 激活布娃娃后布娃娃的初始动作需要指定，让模型动作平滑地从模型动画过渡到布娃娃。
> pBoneWorldTransform 为骨骼矩阵数组的指针，你需要将模型骨骼矩阵计算好然后把指针传给物理引擎，或者你可以直接使用 StudioModelRenderer 计算好的矩阵。
> 例：`gPhysics.SetPose(tempent->entity.index, (float*)m_pbonetransform);`
*  关闭布娃娃效果
`	void(_stdcall* StopRagdoll)(int entityId);`
> 实体复活时，或者其他需要的时候，可以使用此函数关闭布娃娃。再次使用StartRagdoll又可激活布娃娃。

*  更新物理世界
`void(_stdcall* Update)(float delta);`
> 此函数每一帧都要调用，最好放在准备渲染的时候。
> delta 自从上一帧到这一帧过去的时间
*  SetupBones
`void(_stdcall* SetupBonesPhysically)(int entityId);`
entityId 当前渲染实体的Id
> 安插在StudioModelRenderer的`void CStudioModelRenderer::StudioSetupBones ( void )`函数的最后面。
> 此函数会根据当前渲染实体的布娃娃控制器状态来控制模型的动作。
> 
* . 控制器更换宿主
`void(_stdcall* ChangeOwner)(int oldEntity, int newEntity);`
 oldEntity 死亡玩家的实体
 newEntity 新放置的实体
> 使用此函数将旧实体的布娃娃控制器交给新实体。
> 因为在有些mod中，玩家实体死亡过了一会以后将会在死亡的实体位置创建一个模型实体，这时的尸体就由这个新创建的模型实体来表示了，而玩家实体可能已经复活或者有其他的任务了，不在停留在原地。这个尸体就不再属于原来的玩家实体，所以得通过此函数将控制器转交给新实体。
> 注意：因为旧实体的控制器已移交给新实体，所以旧实体不再有控制器。如再需要，得重新为旧实体创建。

## 三、为模型提供额外的数据
简单地来说就是你得告诉布娃娃系统你的模型中哪个骨骼是头，哪个骨骼是大腿……系统才能给你的模型创建碰撞体。
指定关键骨骼让物理系统为您的模型创建布娃娃（只支持类人模型）。
首先使用 sdk\tools 文件夹下的HLMV打开模型，点击下方的Bones栏，你会看到一个下拉菜单。这个下拉菜单里的内容就是这个模型的所有骨骼索引和名称，格式为Bone[索引]=[骨骼名]。

![HLMV骨骼](https://img-blog.csdnimg.cn/20201002134814386.png)

点击某一项你就能看到这个骨骼在模型中的位置了。待会你就可以观察头/手/脚的骨骼索引是多少了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002140036986.png?x-oss-process=image,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI3NzkzODU=,size_16,color_FFFFFF,t_70#pic_center)

接下来在您的 Mod 文件夹里创建 phydata 文件夹，再在 phydata 里创建RagdollBone.json ，里面的内容是为每个模型指定它的关键骨骼。RagdollBone.json 模板：
```json
[
  {
    "Key": "leet",
    "Value": {
      "Head": 0,
      "Spine": 0,
      "Pelvis": 0,
      "LeftArm": 0,
      "LeftElbow": 0,
      "LeftHand": 0,
      "LeftHip": 0,
      "LeftKnee": 0,
      "LeftFoot": 0,
      "RightArm": 0,
      "RightElbow": 0,
      "RightHand": 0,
      "RightHip": 0,
      "RightKnee": 0,
      "RightFoot": 0
    }
  },
  {
    "Key": "gordon",
    "Value": {
      "Head": 27,
      "Spine": 3,
      "Pelvis": 1,
      "LeftArm": 9,
      "LeftElbow": 11,
      "LeftHand": 13,
      "LeftHip": 35,
      "LeftKnee": 37,
      "LeftFoot": 39,
      "RightArm": 10,
      "RightElbow": 12,
      "RightHand": 14,
      "RightHip": 36,
      "RightKnee": 38,
      "RightFoot": 40
    }
  }
]
```
格式为 Key：模型文件名不带文件后缀，Value里的数字为该骨骼在模型查看器里看到的索引。
注意，Value里的字符串是骨骼名，并不一定和模型的骨骼名一一对应，但是也有一些参考，比如LeftArm在模型里的名字是Bip01 L Arm1。这时候有人问了，为什么Bip01 L Arm不是呀？所以还有一个参考点，就是需要您查看骨骼的位置是不是大致吻合。下面第二张图标注了这些骨骼在人类模型的大致位置。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002140136324.png#pic_center)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201005173218163.png?x-oss-process=image,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTI3NzkzODU=,size_16,color_FFFFFF,t_70#pic_center)


给出一般的编辑RagdollBone.json的方法：
1. 找到你需要有布娃娃效果的模型
2. 用sdk\tools下的HLMV打开
3. 打开RagdollBone.json，复制模板。
4. Key 改为你的模型文件名不带后缀。
5. 去 HLMV 找到 Value 中的每一个骨骼的索引填上，比如 Head 就找到模型中大概在脖子上的骨骼，一般也叫Head，但也可能不叫，看美术做模型时的心情。Spine就找在模型腹部的很可能叫Spine1的骨骼，以此类推。
模板中举了有两个模型的例子，你可以添加更多。注意格式。不要忘记两个模型之间的逗号。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210313220330294.png)

编辑完成以后记得保存。之后在游戏中系统就会根据这些给出的数据计算碰撞体和约束，以及在布娃娃状态时物理引擎接管动画的控制。

