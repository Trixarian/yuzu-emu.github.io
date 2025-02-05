+++
date = "2023-11-16T12:00:00-03:00"
title = "Progress Report October 2023"
author = "GoldenX86"
forum = 942091
+++

Hello yuz-ers! This past month, we got a plethora of GPU fixes, support for new applets, a lot of work poured into the Android builds, some interesting news of the future, and more. Let’s get to it!

<!--more--> 

## Wowie Zowie!

A new Mario game! And it’s an excellent one at that.
`Super Mario Bros. Wonder` joins the fray of side-scrolling Mario games and refines the genre's gameplay with its new and *wonderful* levels.

{{< imgs
	"./wonder1.png| Such a pretty game (Super Mario Bros. Wonder)"
  >}}

The game didn’t boot at release due to incorrect used memory reporting in the kernel.
Thankfully, [byte[]](https://github.com/liamwhite) quickly found the culprit. It was a {{< gh-hovercard "11825" "single-line change!" >}}

{{< imgs
	"./wonder2.png| Hope your platforming skills are up to par (Super Mario Bros. Wonder)"
  >}}

However, this wasn’t enough to get the game in a playable state.
Depending on where you are in the game, `Super Mario Bros. Wonder` internally switches between double and triple buffered VSync presentation modes.
This causes the number of images it has available for presentation to frequently change.


yuzu wasn’t ready for this due to a misunderstanding of how nvnflinger works.
On Android, SurfaceFlinger (the OG Flinger) can free buffers that are beyond the maximum count a program has allocated, but nvnflinger (the Switch's fork) is never supposed to free any buffers unless the program requests it.
Maide made a {{< gh-hovercard "11827" "few presentation code changes" >}} to support this behaviour, and players are now set to grab those fun Wonder Flowers!

{{< single-title-imgs
    "Classic Mario (Super Mario Bros. Wonder)"
    "./wonder3.png"
    "./wonder4.png"
    >}}

I bet you didn’t expect to see the sole input change of the month here ― yet here we are.
`Super Mario Bros. Wonder` *loves* vibration, to a point of saturating the old implementation when using HD Rumble on Nintendo controllers (Joy-Cons, Pro Controllers).
This is because waiting for the controller to reply takes time, more than the game would have the patience for, leading to noticeable and bothersome vibration stuttering. That’s right, ASTC, you’re not the only one in town causing stutters.
By {{< gh-hovercard "11852" "making vibration calls asynchronous," >}} vibration is, while not entirely solved, now *much* more pleasant.

## The GPU changes

Undefined behaviour: the formal way to say “here be dragons”.
It’s a good practice to avoid dragons ― I mean, undefined behaviour ― in your code, especially when dealing with a complex graphics API like Vulkan.

Remember our explanation of `depth stencils` [back in August](https://yuzu-emu.org/entry/yuzu-progress-report-aug-2023/#more-gpu-changes)? You may wish to reread that, as it provides useful context to what we will talk about next.

When a game is using a depth buffer, it is usually drawing into a busy 3D scene while taking advantage of a hardware-accelerated process called depth testing.

During depth testing, the GPU hardware determines if a pixel is visible or hidden (occluded) by another pixel. This is decided by their depth values.
The depth buffer tracks how far away each stored pixel is from the camera.
If a rendered pixel is further away than what has already been drawn on the scene, then the pixel is discarded; if it is closer, then it is kept and the colour buffers are updated.
Typically, the depth buffer is also updated and written to, in this case, to store the depth of the new, closer object.

It is possible for a game to use depth testing alone, and turn the actual writes to the depth buffer off for specific elements, and many games do this when rendering partially transparent objects.
However, the opposite is not allowed by graphics APIs like Vulkan ― hardware designs require depth testing to be enabled in order update the depth buffer.
yuzu’s masked clear path for depth/stencil buffers has a shader which updates the depth buffer, and so enables depth writes, but forgot to also enable depth tests.
Most of the time, this worked by coincidence, as the game was enabling depth tests and yuzu was not clearing this state.
However, not all games enabled them, and without depth tests, games like `Super Mario 64`, part of `Super Mario 3D All-Stars`, can’t properly render the face of a certain character (I *think* his name is in the title of the game.)

{{< single-title-imgs-compare
	"Wonder who he is (Super Mario 64)"
	"./depthbug.png"
	"./depthfix.png"
>}}

Thanks to {{< gh-hovercard "11630" "the work done" >}} by [Maide](https://github.com/Kelebek1), Mr. Mario Mario renders correctly now.

Not stopping there, Maide’s dragon-hunting continued for another pull request.
One advantage of using the standard Vulkan Memory Allocator (VMA for short) is how it can help sanitise code.

VMA will raise asserts if things are wrong somewhere.
In this case, we were accidentally marking a device-local buffer we intend to use exclusively in VRAM as CPU mapped.
VMA is very clear here: device local buffers should not be allocated as mapped because they are outright *not* intended for CPU access.
Making it happy {{< gh-hovercard "11734" "has soothed another dragon." >}}

Thanks to users' reports, [Blinkhawk](https://github.com/FernandoS27) managed to figure out why the new query cache was leaking memory in many games, including `The Legend of Zelda: Tears of the Kingdom`.
After {{< gh-hovercard "11646" "some slight tweaking," >}} RAM consumption is put in its place.

Starting a campaign to combat holes in yuzu's format support, [Squall-Leonhart](https://github.com/Squall-Leonhart) has been working on implementing some of the more obscure format conversions, like  `D32_SFLOAT`.
For this particular depth format on Vulkan, it can {{< gh-hovercard "11677" "now be converted" >}} to `ABGR8_UNORM` when the game needs this behaviour.
Combined with {{< gh-hovercard "11716" "adding support for" >}} the `Z32`, `FLOAT`, `UINT`, `UINT`, `UINT`, `LINEAR` variants in the internal format table, this work solves rendering issues in games like `Disney Speedstorm` and `Titan Glory`.

{{< single-title-imgs-compare
	"Look, the most powerful mouse in the world (Disney Speedstorm)"
	"./disneybug.png"
	"./disneyfix.png"
>}}

Some games also make aliases of images in the D32 depth format.
Since a similar limitation with format conversion was present here too, `ARGB8_SRGB` and `BGRA8_UNORM`/`BGRA8_SRGB` {{< gh-hovercard "11795" "can now be converted to" >}} `D32_SFLOAT` to provide proper compatibility.

{{< imgs
	"./gothic.png| Aged graphics have this feel of nostalgia (Gothic)"
  >}}

Continuing with this streak, Maide {{< gh-hovercard "11688" "implemented" >}} the `X8_D24` depth format, allowing `A Sound Plan` to start rendering.
However, more work is needed to make this game properly playable.

Robustness is a feature Vulkan provides that lets developers handle invalid memory accesses in a cleanly defined way.
This can help prevent the application from crashing or ~~summoning dragons~~ invoking undefined behaviour when some part of the code tries to access memory out of bounds.

For some reason, either Maxwell and Pascal NVIDIA GPUs have broken robustness support on uniform buffers, or yuzu’s codebase makes a wrong assumption somewhere (most likely the latter). As a result, those two NVIDIA GPU generations (GTX 750/GTX 900/GTX 1000 series) suffer from oversized graphics on `Crash Team Racing Nitro-Fueled`, due to the game accessing memory out of bounds in the shader.
{{< gh-hovercard "11789" "Manually clamping out-of-bounds buffer reads to 0" >}} on the affected GPU architectures solves the issues. We are also now investigating what causes this problem in the first place.
Maide gets to play detective yet again, dear Watson.

{{< single-title-imgs-compare
	"This GTX 1050 needed a diet (Crash Team Racing Nitro-Fueled)"
	"./ctrbug.png"
	"./ctrfix.png"
>}}

Maide also fixed a hidden issue with the resolution scaler.
Images were being {{< gh-hovercard "11744" "marked as rescaled," >}} even if the resolution scaler was not in use (running at 1x). 
This led to a slight additional overhead, and rarely, some assertion failures.
While no game bug was known to be caused by this, it’s good to have preemptive fixes for once instead of just reactionary ones.

In the meantime, Maide has also been removing image alias bits for all image attachments in an effort to allow the drivers to use more memory optimizations.
This pull request also includes {{< gh-hovercard "11747" "some other minor fixes" >}} with it.

By {{< gh-hovercard "11775" "implementing" >}} the first and subsequent draw commands for vertex arrays, `Super Meat Boy` finally renders correctly! No more black screens!

{{< imgs
	"./smb.png| Well done (Super Meat Boy)"
  >}}

And now, one for the Linux gang.
[v1993](https://github.com/v1993) tested and {{< gh-hovercard "11786" "re-enabled CUDA video decoding" >}} on Linux, allowing better video decoding performance for NVIDIA users (running the proprietary driver, of course).
We previously disabled CUDA by default because it can fail on systems running both a dedicated NVIDIA GPU and a dedicated AMD GPU (iGPUs are fine), a decently rare configuration that only a few people, like your writer, actually ever use.

For those few users running mixed hardware vendors on your systems, please manually  select “CPU Video Decoding” if you’re affected by video decoding issues now. 
This was the previous default behaviour.

We received user reports of crashes occurring when grabbing a Grand Star in `Super Mario Galaxy`, as part of `Super Mario 3D All-Stars`.
byte[] found that the problem is in how the Vulkan scheduler incorrectly flushes data.
The solution? [Use a lock](https://www.youtube.com/watch?v=SNgNBsCI4EA). 
And if that doesn’t work, {{< gh-hovercard "11806" "use more locks." >}}
Happy star hunting!

To improve OpenGL support even further, [Epicboy](https://github.com/ameerj) returns.
First, he found that the `shfl_in_bounds` variable, which is used to track data used in compute shaders, could result in undefined behaviour when threads were inactive and return invalid results. 
{{< gh-hovercard "11847" "The solution" >}} was to move the `shfl_in_bounds` check after the `readInvocationARB` function, which requires all threads to be active, to avoid ~~dragons~~ undefined behaviour. 
This fixed some graphical corruption issues in unit tests, which should lead to fixes in real games too.

Next, a simple gift from epicboy: {{< gh-hovercard "11904" "force enabling" >}} `Threaded optimization`, an NVIDIA-specific OpenGL optimization that enables the use of a new separate CPU thread for graphics rendering.
This is a solid performance boost for those running OpenGL with NVIDIA hardware.
And for those asking, yes, Vulkan allows the use of a separate thread for rendering too ― but we do it on an API level, instead of a driver setting, so all GPUs can benefit.

Maide found a problem occurring with compute shaders when they were triggering invalidations in the buffer cache.
yuzu has a lot of code to track the sizes of buffers used by the games.
Consider a game using two buffers: the first, with an address range from 1 to 2, and another with a range of 3 to 5. They don't overlap, so there is no issue.
Then, after the game runs for a while, a buffer requiring a range of 1 to 5 is used.
The previous two buffers would be considered to overlap it and will be deleted. 
Then, the old buffer data from the two overlaps gets moved to the new third buffer.
While this was already working for graphics-related buffers, it didn’t correctly consider compute buffers.

{{< gh-hovercard "11859" "Adding the missing loop" >}} to fix this behaviour, problems ranging from minimal graphical issues to completely broken game logic are potentially solved.
You know what they say: with compute, the sky's the limit. Just ask AI developers.

Another optimization Maide implemented affects how buffers are handled after they are successfully deleted.
The previous method would unnecessarily create several copies, wasting resources.
By {{< gh-hovercard "11683" "removing one synchronisation step," >}} only the exact amount of needed copies are used.

Now let's discuss a long-standing issue: some 2D games were flipped on AMD, Intel, and Android GPUs.
Games can use different APIs to run on the Switch. 
Nintendo allows the use of Vulkan, OpenGL, and their proprietary API, NVN. 
This poses a problem for emulation on Vulkan, since [only NVIDIA GPUs](https://vulkan.gpuinfo.org/listdevicescoverage.php?extension=VK_NV_viewport_swizzle&platform=all) support the viewport swizzle extension (which allows for transforming viewports).

To allow other GPU vendors to render properly in most cases, a fallback implementation was made to handle the case of vertical flips, for use by OpenGL games.
A tiny error in its implementation made it unable to correctly track invalidations of the viewport flip state, resulting in garbled graphics in several games.
byte[] {{< gh-hovercard "11893" "solved" >}} that issue with a one-line change, making games like `Stardew Valley`, and `Streets of Rage 4`, finally render properly on non-NVIDIA hardware.
No more mirrors needed!

{{< imgs
	"./sv.png| THE cozy farming game (Stardew Valley)"
  >}}

While investigating this, a similar but different issue came to light.
Many OpenGL games (that is, games using OpenGL to render, not yuzu rendering on OpenGL) ask for a 1920x1080 framebuffer, regardless of whether the game is running in handheld or docked mode.
The game then simply moves and resizes the region it’s rendering to inside that 1920x1080 buffer, like moving a small box inside a bigger box, if you will.
In the final step, the image is flipped and sent to the bottom of that 1080p render target.
yuzu was incorrectly rendering only the top of the render target due to how it used to calculate that flip in the final pass.

{{< gh-hovercard "11894" "Adjusting" >}} this behaviour made `Tiny Thor` render correctly in handheld mode.

{{< imgs
	"./tt.png| To Valhalla (Tiny Thor)"
  >}}

And to close the graphics section, by following this chain of events, the previous pull request helped spot a bug in our Vulkan presentation that led to `Arcaea` being incorrectly cropped in handheld mode.
95 lines of {{< gh-hovercard "11896" "cropping-behaviour code changes" >}} later, and byte[] solved the issue.

{{< imgs
	"./arcaea.png| Good art style (Arcaea)"
  >}}

## Android changes

This month the Android build got not only significant UI changes to improve quality of life, but device compatibility was greatly improved for Adreno users.
Here’s the full list!

- A new {{< gh-hovercard "11649" "GPU driver manager" >}} was developed by [t895](https://github.com/t895), allowing the listing of multiple drivers, useful for quickly switching between proprietary Qualcomm or Mesa Turnip releases, or several versions of each. Due to the beta status of Turnip drivers and the immature code of Qualcomm drivers, the latest release is not always the best.

{{< imgs
	"./gpu.png| For all your driver-switching needs!"
  >}}

- byte[] solved a {{< gh-hovercard "11656" "crash related to surface recreation," >}} which could be triggered by simply rotating the device.
- byte[] solved an issue affecting some Android devices that ship an {{< gh-hovercard "11876" "outdated Vulkan 1.1 loader" >}} instead of the current latest 1.3, causing the device to report older features than the driver in use actually supports. This resulted in specific Adreno 600 and 700 devices crashing at boot when using any Mesa Turnip driver version. Forcing the correct Vulkan 1.3 features the driver supports solves the issue. We’ll expand on this at the end of the article.
- t895 implemented a {{< gh-hovercard "11909" "home settings menu grid" >}} for devices with bigger screens and/or higher DPIs. This should please our tablet and foldable users. Enjoy!

{{< imgs
	"./wide.png| Landscape lovers rejoice"
  >}}

- byte[] {{< gh-hovercard "11910" "fixed another case" >}} where yuzu would fail to recreate the surface on screen rotations.
- t895 moved the {{< gh-hovercard "11915" "game list loading process to a separate thread" >}} to reduce stuttering when opening yuzu. The process still takes a similar amount of time, but the perceived smoothness is very welcome.
- t895 solved an issue that caused the {{< gh-hovercard "11916" "touch buttons overlay to get stuck" >}} while drawing the in-game menu from the left side.
- While waiting for a controller settings menu, t895 fixed a bug that caused {{< gh-hovercard "11925" "all controller input to move to player 2" >}} on some devices, blocking users from playing most games. Devices with integrated controllers should have a much better experience now.
- And finally, following the [recent changes](https://yuzu-emu.org/entry/yuzu-progress-report-sep-2023/#of-miis-and-applets) in the desktop version, t895 added a {{< gh-hovercard "11931" "menu to access the currently supported applets," >}} Album and Mii editor, along with the Cabinet applet to manage amiibo data. Wii think you will have fun!

{{< single-title-imgs
    "We hope to expand this selection in the future"
    "./cabinet1.png"
    "./cabinet2.png"
    >}}

{{< single-title-imgs
    "Wii want to play"
    "./mii.png"
    "./album.png"
    >}}

The settings menu was also reorganised:

{{< imgs
	"./settings.png| Hope it’s more convenient now"
  >}}

## UI and Applet changes

After a rocky start, thanks to the early work of [roenyroeny](https://github.com/roenyroeny), [boludoz](https://github.com/boludoz), and [FearlessTobi](https://github.com/FearlessTobi), we now have proper {{< gh-hovercard "11705" "shortcut creation" >}} support for Windows too!

To access this feature, simply right click a game in yuzu’s game list, select Create Shortcut, and pick if you want it on your desktop or the applications section of the start menu. This allows you to start games with a quick start menu search or even from a pin in the menu/taskbar if you so choose.

{{< imgs
	"./shortcut.png| Populate that taskbar"
  >}}

Helping improve this, [german77](https://github.com/german77) made the required changes to {{< gh-hovercard "11740" "save multiple resolutions per icon," >}} making smaller sized desktop icons much more readable than before.

{{< imgs
	"./icons.png| Scaling images down doesn’t always look the best"
  >}}

[DanielSvoboda](https://github.com/DanielSvoboda) made several changes in file system handling to {{< gh-hovercard "11749" "improve directory path detection" >}} for the shortcuts, making them far more usable and stable.
Thank you!

Work on improving user experience (UX) is always welcome, it is your writer’s belief that UX is as important as proper functionality, and should never be ignored.
To improve the quality of life of people playing with multiple controllers, [flodavid](https://github.com/flodavid) changed the behaviour of how users interact with the {{< gh-hovercard "11779" "number of connected controllers" >}} in the Controls settings.
Users can now more intuitively click the green lights at the bottom to select how many players/controllers they would like to be active.
Thank you!

{{< imgs
	"./controls.png| Epic Smash sleepover!"
  >}}

Another welcome UX addition is by [Macj0rdan](https://github.com/Macj0rdan), who implemented {{< gh-hovercard "11903" "a quick control for game volume" >}} with the mouse wheel when the pointer is placed over the volume button in the UI, removing the need to click it and drag a small slider.
Thank you!

Continuing his work on applet support, german77 implemented the `SaveScreenShotEx0` {{< gh-hovercard "11812" "service method and its variants," >}} allowing users to take captures from within games themselves instead of globally with the screenshot hotkey.
This works for games like `Super Smash Bros. Ultimate` ― however, note that screenshot editing is not available yet.
Homework for later!

{{< imgs
	"./smash.png| Save your best moments (Super Smash Bros. Ultimate)"
  >}}

german77 also {{< gh-hovercard "11892" "implemented the" >}} `SaveCurrentScreenshot` method, allowing users to take in-game screenshots in `Pokémon Scarlet/Violet` with its latest update installed.
Happy selfie shooting!

{{< imgs
	"./poke.png| Feeling cute, might capture a shiny later (Pokémon Scarlet)"
  >}}

One of the changes german77 introduced broke the `Find Mii` stage in `Super Smash Bros. Ultimate`.
Since it’s not a popular stage, even among the Smash community, we didn’t notice this issue, and no one reported in our [Discord server](https://discord.gg/u77vRWY), [forums](https://community.citra-emu.org/c/yuzu-support/14), or [GitHub bug report page](https://github.com/yuzu-emu/yuzu/issues/new/choose) ― we only found out thanks to complaints on Reddit. This is a reminder to you, the users, please try to report issues in the proper channels! Doing so ensures we see the problems you are having and can work to fix them.

yuzu is a large project with many “black box” areas. There will be bugs, many thousands of bugs, that our team may never encounter. Your voice is so important, we hate to see any bug reports slip through the cracks outside of our channels.

With that out of the way, by {{< gh-hovercard "11822" "creating random Miis" >}} with names, german77 solved the issue.

german77 also {{< gh-hovercard "11846" "expanded the character limit of cheats" >}} to more than 64 characters.
Now you can cheat to your heart's content.

You know our developers spend a lot of time opening and closing yuzu when byte[] can accurately measure that 10% of shutdown crashes are caused by the game list.
He found that the issue was in how Qt deals with messages from objects that were destroyed or disconnected (like stopping the emulator while the game list is loading).
By {{< gh-hovercard "11846" "changing the behaviour" >}} of how the game list is reported to those events, another source of shutdown crashes has been defeated.

## Kernel, CPU, and file system changes

byte[] has been having a lot of “fun” lately fixing and implementing kernel changes.
First off, he {{< gh-hovercard "11686" "fully implemented transfer memory," >}} fixed {{< gh-hovercard "11766" "incorrect page group tracking," >}} {{< gh-hovercard "11914" "updated the implementation" >}} of KPageTableBase, and has now {{< gh-hovercard "11843" "nearly completed" >}} the entire KProcess implementation!

As preliminary work for NCE support coming in the near future, byte[] implemented {{< gh-hovercard "11718" "native clock support" >}} for arm64 devices running on Linux or Android.
There is no support for ARM Windows devices for now, as none have bothered to include a Vulkan driver yet.

The kernel was updated to reflect changes made in {{< gh-hovercard "11748" "firmware version 17.0.0," >}} ensuring support for future games.

v1993 {{< gh-hovercard "11772" "solved some warnings" >}} that were spamming our build logs ― namely, using `std::forward` where appropriate, and qualifying `std::move` calls.
This should solve build issues for those experimenting with Darwin build targets.

By user request, byte[] {{< gh-hovercard "11774" "further improved the build performance of RomFS mods" >}} by getting rid of some unnecessary object copies.
This also fixed a file handle leak, which now allows modders to edit mod files after stopping emulation, helping them work faster on those *juicy and delicious* game mods.

## Audio changes

Thanks to user reports, our audio connoisseur, Maide, found that `Ancient Rush 2` would crash at the end of the first developers screen.
{{< gh-hovercard "11735" "Clearing the DSP buffer" >}} after each execution fixes the issue.

{{< imgs
	"./ar2.png| Diggy Diggy Hole! (Ancient Rush 2)"
  >}}

Speaking of audio, byte[] delivered another resounding victory in the shutdown department by {{< gh-hovercard "11778" "fixing a deadlock" >}} in the audio renderer.

## Hardware section

We have some sad news for the old Red Team guard, a warning for the Green Team, and very good news for Adreno droids.

### NVIDIA, our fault this time

The 545 and 546 series of drivers have solved the high VRAM usage crashes we reported this last month, but users are reporting new crashes in games with these drivers.
Reverting to the 53X series of drivers solves the problem, but this time, it's not NVIDIA’s fault!
Mesa also has many new crashes with the 24.0.0 release, and having the top two drivers crashing under the same conditions is not a coincidence.

We found that the problem is in how we are recompiling shaders ― the types of some variables are mismatched.
The fix will take some time to implement, so for NVIDIA and Mesa users experiencing crashes in games like `Bayonetta 3`, we suggest not updating your drivers for the time being.

### AMD, giving a last hurrah to Polaris and Vega

The time has come. The last 2 remnants of the GCN architecture are on their way to be discontinued.
AMD started releasing [split drivers](https://www.anandtech.com/show/21126/amd-reduces-ongoing-driver-support-for-polaris-and-vega-gpus) for those products, which run outdated Vulkan driver branches compared to RDNA and newer hardware.
[The news](https://www.phoronix.com/news/Mesa-24.0-Faster-RADV-Vega) of AMDVLK, the official Linux AMD driver, killing support for these products means no new Vulkan drivers will be available.

This doesn’t mean the show is over for their owners. 
For the time being, no new change breaks compatibility with the cards, and Linux Mesa drivers like RADV will continue to provide support, most likely extending it past what the Windows drivers report as supported (as is usually the case with Mesa).

But for those stuck on Windows, this is the last ride.

GCN4.0, GCN5.0, you weren’t the most efficient cards, but you gave us good value in the worst moments, *years* of amazing gameplay, and great FineWine moments.
We salute you and thank you for your impeccable service, few GPU architectures leave the stage with such a round of applause.

o7

### Turnip, a very quickly improving work-in-progress

yuzu uses a single codebase for all its releases, upstream/master/main, however you prefer to call it, Mainline, Early Access, and Android all start from there.
When improvements to the codebase are added, they eventually reach all releases, Android included.

We recently added improved support for `occlusion queries` (part of project Y.F.C.) to Android to increase performance and accuracy on all devices. But occlusion query support on Turnip drivers with Adreno 725 and 730 GPUs was not working correctly, and this took us a very long time to find.
Users experienced crashes that forced them to remain on outdated [GitHub](https://github.com/yuzu-emu/yuzu-android/releases/) versions, and we needed to stall a new Play Store release until the problem was investigated and properly solved.

The issue was found, reported, and resolved by Mesa in record time in their current Adreno 700 branches, which then driver packagers like [K11MCH1](https://github.com/K11MCH1/AdrenoToolsDrivers/releases) use to build packages Qualcomm users can load on emulators.

For this reason we *strongly* recommend Adreno 730 and Adreno 725 users to update to the latest [Release X](https://github.com/K11MCH1/AdrenoToolsDrivers/releases/tag/24.0.0-R-X) driver, which not only fixes the crashes caused by occlusion queries, but also the lack of support for the Adreno 725 and some variants (yes, there are several) of the Adreno 730. 
It isn't only desktop GPU vendors who love to rename things.

By using this driver, we were able to launch newer Android builds with the latest upstream changes, and all Qualcomm users can safely use the latest GitHub builds if they prefer.

By the way, the GitHub builds are now properly signed. Rejoice, as you can now update from one to the next without needing to uninstall first.
If you want to test experimental and potentially unsafe (but maybe faster) changes before Play Store updates, it’s now much easier.

## Future projects

Let’s begin with what most people want to hear about: Project Nice for Android devices.
NCE (Native Code Execution) is progressing very well, but there are still some bugs to iron out. 
Games are becoming not only playable, but also _faster_ on devices with thermal restrictions. Additionally, the time spent loading and closing games has been significantly reduced now!
NCE has helped us understand issues in our CPU emulation in x86_64 too, so expect gains on both fronts.

{{< imgs
	"./smo.mp4| Recorded with a Red Magic 7S Pro (SUPER MARIO ODYSSEY)"
  >}}

No promises on a release date per usual, but as politicians love to say: “We’re working on it.”

{{< imgs
	"./bayo2.mp4| Destroying those touch screen controls! (Bayonetta 2)"
  >}}

Blinkhawk is *suggesting* that your writer informs you he is working on a new project seeking to make the GPU process-agnostic, allowing multiprocess emulation within the GPU. 
No fancy names this time, we’ve simply been calling this “Multiprocess” internally.
Multiprocess is a mandatory step towards UMA support, which will provide huge gains for iGPUs and SoC users, as well as reducing RAM consumption.

That’s all folks! Apologies for the delay, university homework and finals are killing your writer.
Thank you for reading until the end. See you next time!

&nbsp;
{{< article-end >}}
{{< imgs-compare-include-end >}}
{{< gh-hovercard-include-end >}}
