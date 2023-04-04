| English | [简体中文](README_CH.md) |

# Description 

This game has been out for over a year now, and the developer disappeared in the middle of last year, leaving behind a game full of bugs. I think it's a shame, so I decided to take matters into my own hands and try to fix some of the really nasty bugs that affect the game experience by decompiling and recompiling the IL files. Although I haven't learned C#, nor do I have any game programming experience, fortunately my main programming language is C++, which has many similarities in syntax with C#. So I don't feel much pressure when reading the game's code. Currently, it's known that this patch has successfully fixed the issue of disappearing laser weapon projectiles. In theory, it should apply to all weapons that have the problem of projectiles disappearing, but due to my limited time, I couldn't test them all. If I have free time in the future, I will try to fix more bugs and update them in this guide.

# Download Link

Please check [the Releases page of this GitHub repository](https://github.com/DexArson/Mechajammer-unofficial-patch/releases).

# How To Use

Copy the DLL file you downloaded to `your_game_installation_folder\Mechajammer_Data\Managed`

If you are not interested in the analysis of the code of this game and the cause of bugs, you can close this page now without browsing the following parts.

# Bug Analysis Process

I suddenly realized that my the projectile of my laser pistol disappeared when fired, while other guns were working fine. Obviously, this was a bug. As I hadn't picked up lazer gun ammo for a long time, I didn't know when the laser gun started having this issue. My first suspicion was that it was triggered by certain ability I had upgraded. Since the beginning of the game, I had only invested skill points in hacking, lockpicking, laser weapons, and grace. Obviously, hacking and lockpicking had nothing to do with laser guns, so I focused my suspicions on the latter two.

I started by locating the game's save file and Modifying it. Modifying the save file of this game was a piece of cake - it wasn't even stored in binary, but rather in plain text. I individually modified the Grace and Laser skills and found that the problem was with Grace. When I lowered Grace from 3 to 2, the laser gun started firing normally again. However, I didn't want to keep my Grace at 2 forever because of this bug, so I started decompiling the game files in the hope of pinpointing the problem from the code level.

First, I located the part of the game code related to the dice-rolling mechanism and made some modifications and tests to the part related to Grace, but the problem persisted. 

Then, I started looking into the code of the "turn" mechanism, which is also affected by Grace, and examined the code related to it. I found the following code in the `UpdateStats` function of the `CharacterInfo` class:

```cs
this.reloadTurns = 3 - ((int)this.Grace + (int)this.GraceMod);
this.recoveryTurns = 3 - ((int)this.Grace + (int)this.GraceMod);
if (this.recoveryTurns < 0)
{
	this.recoveryTurns = 0;
}
```

Obviously, when my Grace skill points reached 3, it caused `this.reloadTurns == 1` and `this.recoveryTurns <= 0` (`this` refers to the `CharacterInfo` object, and these two are its member variables). reloadTurns obviously affects reloading, which has nothing to do with firing, so I tried modifying `recoveryTurns` to force it to 1 when `this.recoveryTurns <= 0`. After opening the game again, the laser pistol could fire normally again.

But simply allowing the laser gun to fire normally is not enough for me. After all, from the code perspective, the game developer set this skill to have a minimum value of 0. Moreover, this is clearly just a superficial manifestation of a deeper issue. If I stop at this point, the unresolved underlying problem may cause other bugs to appear.

I restored the code and tested my other guns: the slug pistol and the plasma pistol (in addition, I had picked up two slug rifles, but I lost them due to item disappering bugs), of course they can work normally as before. I also tested the laser pistol in its burst fire mode, and oddly enough, it was able to hit enemies normally. Only single-shot mode was not working.

Based on the differences between the single shot and burst fire modes, I made a further guess: for unknown reasons such as animation scripts or weapon value settings, the laser gun's "fire" in the current turn only played a firing animation for the character, and the actual firing "behavior", because it exceeded the non-pause time of a turn, did not produce any projectiles in this turn but rather after the next pause ended. Due to the existence of the action recovery turns mechanism in the game, the character may not pause immediately after firing for a short period of time. During this time, the laser bullets can be produced normally. When the action recovery turns is 0, the firing action is executed and the game is immediately paused, which causes some values to be initialized incorrectly, leading to the disappearance of bullets.

Thus, my next focus should be on the firing and projectile generation part. As I am not the developer of this game and only have access to decompiled code through IL files, without the built-in debugging tools, debugging symbols provided by Unity, and code comments, the debugging process may become more complicated. Fortunately, the game has a console window in the lower right corner that can output information, allowing me to output the specific values of some internal variables.

First is the game's turn checker: the `Turns` class:

```cs
public class Turns : MonoBehaviour{};
```

This class inherits from Unity's `MonoBehaviour` class, so it has a function named `Update()` that is called every frame in the game. I found this function and followed its call chain down to the code related to projectile creation and its corresponding class: `AttackLine`. Since its base class is also `MonoBehaviour`, I guess it will be attached to a object inherited from `GameObject`, possibly related to projectiles.

When the player character executes an attack action, an `AttackLine` object is requested from the pool in `AIAction.DoAttack()`, and `AttackLine.StartTrajectory()` is called to initialize some information about the projectile. At the same time, `StartTrajectory()` saves the projectile in `Turns` object, ensuring that it exists throughout its intended lifetime. `AttackLine` will call its own `Update()` method every frame while it exists, constantly calculating the current position of the projectile and checking if it hits any objects with a raycast. If it hits an object, the projectile is destroyed.

I manually added some "debugging information" in `AttackLine.Update()` and its called function `AttackLine.DoUpdate()`, allowing it to output the current position and orientation information of the bullet, as well as the current execution position of the function. Finally, I found that after the laser gun fired, the bullet did indeed appear in the next turn, and its initial position coordinates were obviously an incorrect value: (10000,0,0), which confirms part of my speculation. So the next question to figure out is why the initial position of the bullet was not correctly initialized?

Looking at the overall structure of `AttackLine.DoUpdate()`:

```cs
public void DoUpdate()
{
    if (!GameBrain.boo.turns.turnPaused)
    {
    	if (!this.wpnSet)
        {
            // Initialize the values and set wpnSet to true
        }
        // calculate the trajectory
    }
	this.wpnSet = true;
}
```

it's easy to see its intention: when the `Update()` function of the `AttackLine` object is called for the first time, it sets the value of a boolean member variable `wpnSet` of `AttackLine` to true, indicating that the basic information of the projectile has been initialized. My intuition told me that this might be related to the real cause of the bug. Therefore, I tried to monitor this value during runtime and found that when the projectile position error occurred at (10000,0,0), `wpnSet == true`. This is obviously incorrect based on the intended purpose of this variable. 

The answer is now very clear. <span style="text-decoration:underline;">After the first turn is completed, the game enters a pause state, and at this time the `AttackLine` object starts its first `DoUpdate()` function call. However, note that since the value of `!GameBrain.boo.turns.turnPaused` is `false` at this time, this function call returns before completing its basic information initialization. However! Because the assignment statement `this.wpnSet = true;` exists outside of the brace of `if (!GameBrain.boo.turns.turnPaused)`, this assignment statement will always execute regardless of whether the turn is paused or not. In the subsequent `DoUpdate()` method, the `AttackLine` object assumes that its projectile's basic information has been correctly initialized, and thus begins calculating the trajectory based on this incorrect information. Consequently, it generates the bullet at the wrong location, far away from the intended target, and cannot hit the enemy.</span>

So I edited `DoUpdate` function into this form:

```cs
public void DoUpdate()
{
    if (GameBrain.boo.turns.turnPaused)
	{
		return;
	}
    if (!this.wpnSet)
    {
        // Initialize the values and set wpnSet to true
    }
    // calculate the trajectory

    this.wpnSet = true;
}
```

After entering the game again, I found that the laser gun's projectile could be correctly generated in the next turn!

However, there was still one question lingering in my mind: why could the bullets from the plasma gun and the conventional gun be generated in the same turn, but not the laser gun? This clearly had something to do with the weapon's numerical design. So, I opened the backpack in the game and began browsing through all the weapons' information, looking for something related to turns (I had never looked at their specific information in the game before), and finally, I found a weapon attribute called 'Attack Turns', with values of 0, 1, and 2 for the laser, conventional, and plasma weapons, respectively.

This was a very interesting discovery. I had always thought that the laser had a longer attack time than other weapons, but in fact, it was the shortest. There must be a reason why it was so special that it could only generate bullets in the next turn. After carefully reviewing all the functions in 'Turns', I realized that my previous understanding of the 'turns' in the game had actually overlooked an important detail.

(undone yet)