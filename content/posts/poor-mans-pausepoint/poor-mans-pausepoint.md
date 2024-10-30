---
title: "Poor Man's Pausepoint"
date: 2024-10-29T18:36:51-07:00
draft: false
toc: false
images:
tags: 
  - editor
  - debugging
---


If you're familiar with the Jetbrains Rider IDE and have used it for Unity development then you may be aware of a feature they provide called [pausepoints](https://blog.jetbrains.com/dotnet/2020/06/11/introducing-unity-pausepoints-for-rider/). Similar to normal breakpoints but, unlike them, they do not suspend the application nor give you call stack information. Instead it allows the developer to pause the editor when in play mode just like what happens when the user clicks the pause button in the game view. This means that scene edits can still happen as well as inspect and modify the values for anything spawned in the scene. This workflow can be invaluable for gameplay debugging or even tweaking gameplay data at runtime. In this post I will discuss a poor man's version of pausepoints that can be achieved in Tempest using Visual Studio and what it took to get there.

## Engine Architecture

Lets begin with what it took to get the editor pause functionality working. It first started with the easy part and that was extending the scene viewport toolabr to add a pause button. You may have seen in other posts screenshots of the editor where it had one single play button at the top middle of the scene viewport. There was already a pretty generic codebase supporting that play button so it didn't take much to extend it to add a new pause button and then adding a bit more code to keep track of the pause state for the button so the editor can inform the developer when it is enabled. Tempest editor UI uses the concept of signals to enable event driven execution. The pause button is no exception as it uses a signal that the editor can connect to in order to be notified of when it is pressed and handle the event accordingly. I've added a screenshot below of what that toolbar looks like now and you may notice a third button in there as well which will be covered later on.

![New Buttons](/images/poor-mans-pausepoint/NewButtons.png "New Buttons")

With that in place, we have an easy way to toggle the feature and be able to see its current state by looking at the color of the pause button. Now comes the slightly more difficult part, modifying the editor the support this "paused" mode. Before doing that it's worth discussing the requirements that this "paused" mode should achieve.

* Stop script updates - any gameplay script should not be simulating its logic
* Only transform component system should update - this allows selecting and moving entities in the scene while in paused mode
* Every editor feature works like normal

Those are all the main **must haves** for this mode to be considered working properly. I've already covered the engine loop before so I'll only cover the additions to support the pause mode. At every loop the engine will check to see if it is in pause mode just like the snippet below. It will also adjust the frame's delta time and other time variables if it is paused.

```cpp
FrameInfo info;
info.enginePaused = sceneState == SceneState::Play && enginePaused && !localStepToNextFrame;
if (info.enginePaused)
{
    info.deltaTime = 0.0f;
    info.fixedDeltaTime = 0.0f;
    info.unscaledDeltaTime = 0.0f;
}
else
{
    info.deltaTime = deltaTime * timeModulation;
    info.fixedDeltaTime = msPerUpdate;
    info.unscaledDeltaTime = deltaTime;
}
```

**FrameInfo** is a POD struct with various useful frame specific info that gets passed to pretty all other subsystems in the engine. In the code snippet you will see that the engine will be considered to be in pause mode only when the scene is in play mode and if the pause button has been toggled on (that's what **enginePaused** is). If we are to simulate the next frame in paused mode then we make sure that any time variables will be set to 0 so as to not advance any simulations that are currently in progress.

The scripting engine in Tempest has its own API locked away in its own library so it is very self contained. In order to not simulate any active scripts the engine skips simulating the scripts when the editor is paused similar to the snippet below.

```cpp
if (sceneState == SceneState::Play)
{
    if (!enginePaused)
        ScriptingEngine::OnSimulate(info, context);
    activeScene->OnUpdateRuntime(info, context);
}
```

Again the logic only happens when the active scene is in play mode and the engine has a single function to serve as the entry point to simulate all the active scripts in the scene. By skipping that when the editor is paused then we achieve one of the requirements for the pause mode. Namely stop script updates when paused.

The last thing is that Tempest has the concept of a component system where generally speaking each engine level component has its own system that will handle logic updates and will use multithreading when it makes sense. For instance, the transform component has a system that will handle updating any dirty transform since the last update was done. To meet our second requirement to only update the transform component system, the engine relies on the FrameInfo::enginePaused member variable mentioned above. When that is set to true only the component systems marked as **Very High** priority will get updated and the rest will be ignored. The transform component system is one such system.

With the Tempest codebase having minimal coupling with other parts I was actually able to achieve the third requirement for the "paused" mode definition in having the editor functionality just work as normal while being paused. At this point the groundwork for pausing the editor is complete and fully functional.

Lets talk about the third button next to the pause button. As you may know, Unity has the same kind of button in their editor as well. It is used only when the editor is paused and allows stepping to the next frame while remaining paused after the step. This is super useful when debugging single frame bugs that can be tricky to find with normal means. After implementing the base pausing functionality, there was only a very small edit required to make this work. Like the pause button, the step button uses signals as well to inform the editor when it is pressed. After pressing the editor will flag a boolean to let it know that on the next frame it should step to it instead of remaining paused. After finishing the new frame that boolean is reset to false. What happens when we need to step into the next frame was already shown in the first code snippet above. The **localStepToNextFrame** variable determines if the time variables will be set to 0 or use the actual timing values that would normally be used. That's pretty much all there is to that functionality. Simple yet powerful.

Now that Tempest has a way to pause the editor, we can extend it by adding a few more ways to trigger the pause functionality without having to click the button. An example of this is adding a console command that can trigger the pause so all the API around the console commands can be used to facilitate this. Other examples include adding easy to use functions in the gameplay layer so when certain conditions happen the editor could be paused from code. Think of **Debug.Break** in Unity. At some point I may also add a global keyboard shortcut to toggle the pause state but for now I haven't needed it.

## Pausepoints

Rider offers a custom Unity plugin that offers many convenient features when working with Unity, one of them being pausepoints. The way it works is that it provides a modified breakpoint UI in the IDE so you can use pausepoints in the much the same way you do breakpoints. The workflow is you add a normal breakpoint where you want the pausepoint to be and then right click on it to convert it to a pausepoint. I set out to try to achieve something similar with Tempest but I quickly ran into many issues trying to build a custom IDE plugin for Tempest in Rider as well as Visual Studio. Main issue I ran into was a severe lack of documentation and the few examples available didn't really cover things you would need to know to make something useful. Needless to say after a day spent on this without making significant progress made me decide against making a custom IDE plugin.

I decided to pivot into finding a way to achieve a fake pausepoint with the features available within Visual Studio. Eventually found a workflow that simulates a pausepoint though not as nice as having a custom plugin to facilitate it, hence why I call it the poor man's pausepoint.

To emulate the pausepoint you start by placing a normal breakpoint where you would like to put the pausepoint and then let the editor trigger the breakpoint. Once that happens you can go to the watch window in Visual Studio and call a function to trigger the pause logic. Typically you use the watch window to actually watch variable values but a function call can actually be done. Tempest already has a static function that can trigger the pause logic and that is what is used inside the watch window. When doing that make sure you type the whole namespace chain to call the function instead of just the class name followed by the function name.

```cpp
// Be explicit and use all the namespace names the function lives in followed by class name and function name
Tempest::GameInstance::TriggerEditorPause()
```

Visual Studio also has the **Immediate Window** found in **Debug > Windows > Immediate** and you can do the same thing as in the watch window and type the function you want to call while the application has been suspended due to the breakpoint that was triggered. After typing the function and hitting the enter key you can remove the breakpoint and continue execution. In the next frame you will notice the pause logic has been triggered and you can use the editor like normal.

This gives you the same results that the Rider plugin provides but requires a little extra work on the developer's part. In return you get a valuable feature and it's not a lot more work than what the Rider plugin requires so I wanted to share this. I also focused on using this workflow within Visual Studio but it also works in Rider too if you do the same thing in their equivalent watcher window UI. So you get a consistent workflow across IDEs without having to worry about implementing and maintaining different IDE plugins to provide this feature.