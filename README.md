| English | [¼òÌåÖÐÎÄ](README_CH.md) |

# Description 

This game has been out for over a year now, and the developer disappeared in the middle of last year, leaving behind a game full of bugs. I think it's a shame, so I decided to take matters into my own hands and try to fix some of the really nasty bugs that affect the game experience by decompiling and recompiling the IL files. Although I haven't learned C#, nor do I have any game programming experience, fortunately my main programming language is C++, which has many similarities in syntax with C#. So I don't feel much pressure when reading the game's code. Currently, it's known that this patch has successfully fixed the issue of disappearing laser weapon projectiles. In theory, it should apply to all weapons that have the problem of projectiles disappearing, but due to my limited time, I couldn't test them all. If I have free time in the future, I will try to fix more bugs and update them in this guide.

# Download Link

Please check the Releases page of this GitHub repository.

# How To Use

Copy the DLL file you downloaded to `your_game_installation_folder\Mechajammer_Data\Managed`

If you are not interested in the analysis of the code of this game and the cause of bugs, you can close this page now without browsing the following parts.

# Bug Localization Process

I suddenly realized that my the projectile of my laser pistol disappeared when fired, while other guns were working fine. Obviously, this was a bug. As I hadn't picked up lazer gun ammo for a long time, I didn't know when the laser gun started having this issue. My first suspicion was that it was triggered by certain ability I had upgraded. Since the beginning of the game, I had only invested skill points in hacking, lockpicking, laser weapons, and grace. Obviously, hacking and lockpicking had nothing to do with laser guns, so I focused my suspicions on the latter two.

I started by locating the game's save file and Modifying it. Modifying the save file of this game was a piece of cake - it wasn't even stored in binary, but rather in plain text. I individually modified the Grace and Laser skills and found that the problem was with Grace. When I lowered Grace from 3 to 2, the laser gun started firing normally again. However, I didn't want to keep my Grace at 2 forever because of this bug, so I started decompiling the game files in the hope of pinpointing the problem from the code level.