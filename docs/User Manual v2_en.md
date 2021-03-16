![Insert picture description here](https://img-blog.csdnimg.cn/20201002141256140.png)

@[TOC]

Integrate gsphysics in your mod is very simple, and a simple implementation only requires dozens of lines of code. If there is anything you don't understand, please raise an issue directly without hesitation.
Before we start, we need to understand what GoldsrcPhysics is. GoldsrcPhysics, abbreviated as gsphysics, is a **client** physics system that provides physical effects for games built on the Goldsrc engine. HLMod developers can use only a few APIs in the HLSDK client module. The ragdoll effect provided. [The current version only implements rag dolls]
   
At this time, someone has to ask, CS1.6 is also a Mod of Half-Life1, so can CS1.6 have ragdolls? This is possible. Although we do not have the source code of CS, it is possible to use [metahook](https://tieba.baidu.com/f?kw=metahook&ie=utf-8) to manipulate CS. Change some functions of hook into your own implementation and call the API provided by this system.
   > Note: This tutorial will not teach you which line to call an API. You need to have a certain HLMod development experience. You can call the system's API at the right place to provide you with services based on your game logic.
# How does this system work

# Use in your HLMod
[Demo source](https://github.com/anchurcn/HL1RagdollMod)
## Step one, install the module
1. [Download](https://github.com/anchurcn/GoldsrcPhysics/releases) and unzip gsphysics. It contains SDK and dependent libraries.
2. Place the gsphysics folder in the root directory of Half-Life, at the same level as hl.exe.
3. Make sure that the machine has installed .NET 4.6.1 runtime, the newer version of Win10 comes with it, if not, go to [Official Website Download](https://dotnet.microsoft.com/download/dotnet-framework/net461) . [.NET Framework System Requirements](https://docs.microsoft.com/zh-cn/dotnet/framework/get-started/system-requirements)

![Insert picture description here](https://img-blog.csdnimg.cn/2021031015063845.png)
## Step two, coding
 1. Open your client project
 1. Add the two files physics.h;physics.cpp in the gsphysics/sdk directory to your project
 1. Basic API reference:

The APIs listed below have been tested, and the APIs not listed are tried by yourself.

All APIs of GoldsrccPhysics are declared in the physics.h file. The global variable gPhysics contains all the APIs and is called by `gPhysics.XXXXXXX();`.
> Attention! Before calling the functions in gPhysics, call `InitPhysicsInterface(NULL);` to initialize gPhysics. This function will fill gPhysics with the function pointer, and the parameter msg is NULL, which has no actual effect yet.
* Initialize the system
`Function prototype: void(_stdcall* InitSystem)(const char* modFolder, void* pEngineStudioAPI);`
> Called after the game is loaded to initialize the physics system. It only needs to be called once in the life cycle of Mod dll.
> modFolder is your mod folder name string
> pEngineStudioAPI is a pointer to engine_studio_api_t
> The above information can be found in StudioModelRenderer.h/ and StudioModelRenderer.cpp in HL sdk.
> Example: `gPhysics.InitSystem("valve", &IEngineStudio);`
> After the client dll is loaded, HUD_Init and StudioRenderer::Init will be called once. When I tested it, it was called in StudioRenderer::Init.

* Load the collision body of the map for the physical world
`void(_stdcall* ChangeLevel)(char* mapName);`
> Every time you load the map, you need to call this function to load the bsp world collision body of the current map for the physical world. Calling this function will clear the collision body of the previous map (if any).
> mapName given map name (without suffix name and path)
> Can be called in the HUD_VidInit() function. The engine will call HUD_VidInit once when loading the map. But it will also be called when the resolution is adjusted. Pay attention to check that mapName is not null.

* Create a ragdoll controller for the player entity
`void(_stdcall* CreateRagdollControllerModel)(int entityId, void* model);`
> entityId entity ID
> model pointer to the model used by the entity
> Calling this function will create a ragdoll controller of the given model for this entity within the physical system. Subsequent functions such as StartRagdoll are all operated on this controller. If no ragdoll controller is created for the entity, calling StartRagdoll has no effect.
>In fact, in addition to players, any entity that uses a human model can be used, such as barney, scientist, zombie, etc. in npc. If this entity will have a ragdoll state (or that the entity will die), call this function to create a ragdoll controller for this entity.
>Note: Use this function to change the model binding of the ragdoll controller when the entity changes the model.

* Activate the ragdoll effect for the entity
`void(_stdcall* StartRagdoll)(int entityId);`
> When the entity dies, call this function to activate the ragdoll effect for the entity that has created the ragdoll controller. Next, the model of this entity will not be controlled by the animation system, but will use the physics engine to calculate the motion of the model.
> It can be called when the entity's life value is 0, when the first frame of the death animation is played, or when certain death events are triggered, depending on your game implementation.

* Set the first action of the ragdoll
`void(_stdcall*SetPose)(int entityId, float* pBoneWorldTransform);`
> After activating the ragdoll, the initial action of the ragdoll needs to be specified, so that the model movement can smoothly transition from the model animation to the ragdoll.
> pBoneWorldTransform is a pointer to the bone matrix array. You need to calculate the model bone matrix and pass the pointer to the physics engine, or you can directly use the matrix calculated by StudioModelRenderer.
> Example: `gPhysics.SetPose(tempent->entity.index, (float*)m_pbonetransform);`
* Turn off the ragdoll effect
`void(_stdcall* StopRagdoll)(int entityId);`
> When the entity is resurrected, or when needed, you can use this function to close the ragdoll. Use StartRagdoll again to activate the ragdoll.

* Update the physical world
`void(_stdcall* Update)(float delta);`
> This function must be called every frame, preferably when preparing to render.
> delta The time elapsed since the previous frame to this frame
* SetupBones
`void(_stdcall* SetupBonesPhysically)(int entityId);`
entityId The Id of the currently rendered entity
> Insert it at the end of the `void CStudioModelRenderer::StudioSetupBones (void)` function of StudioModelRenderer.
> This function will control the actions of the model according to the state of the ragdoll controller of the current rendering entity.
>
*. Controller replacement host
`void(_stdcall* ChangeOwner)(int oldEntity, int newEntity);`
 oldEntity The entity of the dead player
 newEntity newly placed entity
> Use this function to hand over the ragdoll controller of the old entity to the new entity.
> Because in some mods, after the player entity has died for a while, a model entity will be created at the location of the dead entity. At this time, the corpse is represented by the newly created model entity, and the player entity may have been resurrected or There are other tasks, so I'm no longer staying in place. This corpse no longer belongs to the original player entity, so the controller must be transferred to the new entity through this function.
> Note: Because the controller of the old entity has been handed over to the new entity, the old entity no longer has a controller. If you need it again, you have to recreate it for the old entity.

## Step three, provide additional data for the model
Simply put, you have to tell the ragdoll system which bone in your model is the head and which bone is the thigh...The system can create collision bodies for your model.
Specify key bones and let the physics system create a ragdoll for your model (only humanoid models are supported).
First use the HLMV under the sdk\tools folder to open the model, click on the Bones column below, and you will see a drop-down menu. The content in this drop-down menu is the index and name of all bones of this model, in the format Bone[index]=[bone name].

![HLMV skeleton](https://img-blog.csdnimg.cn/20201002134814386.png)

Click on an item and you can see the position of the bone in the model. You will be able to observe the bone index of the head/hand/foot later.

![Insert picture description here](https://img-blog.csdnimg.cn/20201002140036986.png)

Next, create a phydata folder in your Mod folder, and then create RagdollBone.json in phydata. The content inside is to specify its key bones for each model. RagdollBone.json template:
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
The format is Key: the model file name does not have a file suffix, and the number in Value is the index of the bone seen in the model viewer.
Note that the character string in Value is the bone name, which does not necessarily correspond to the bone name of the model, but there are some references. For example, the name of LeftArm in the model is Bip01 L Arm1. At this time, someone asked, why isn’t Bip01 L Arm? So there is another point of reference, that is, you need to check whether the positions of the bones roughly match. The second icon below indicates the approximate location of these bones in the human model.

![Insert picture description here](https://img-blog.csdnimg.cn/20201002140136324.png#pic_center)

![Insert the picture description here](https://img-blog.csdnimg.cn/20201005173218163.png)


Give a general way to edit RagdollBone.json:
1. Find the model you need to have a ragdoll effect
2. Open with HLMV under sdk\tools
3. Open RagdollBone.json and copy the template.
4. Change the Key to your model file name without suffix.
5. Go to HLMV to find the index of each bone in the Value. For example, Head will find the bone in the model that is probably around the neck. It is usually called Head, but it may not be called. It depends on the mood when the artist is making the model. Spine looks for the bone that is probably called Spine1 in the abdomen of the model, and so on.
There are two examples of models in the template, you can add more. Pay attention to the format. Don't forget the comma between the two models.

![Insert picture description here](https://img-blog.csdnimg.cn/20210313220330294.png)

Remember to save after editing. Then in the game, the system will calculate the collision body and constraints based on the given data, and the physics engine will take over the animation control when in the ragdoll state.
