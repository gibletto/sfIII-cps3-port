Street Fighter III New Generation for Windows
=============================================

A native port of the arcade game (not an emulator). This zip contains NO game data: you need your own copy of
the arcade ROM set.

https://github.com/gibletto/sfiii-cps3-port

1. Add the ROM set
------------------
Put the MAME ROM set sfiiin.zip (non-merged; the older set with one file per SIMM works too) in the roms
folder, as it is. Nothing to convert:

    roms/sfiiin.zip

2. Play
-------
Run sfn.exe. The settings and high scores are saved in roms/eeprom.bin.

    Arrows            move
    A  S  D           light / medium / heavy punch
    Z  X  C           light / medium / heavy kick
    5 then 1          insert a coin, then start (6 and 2 for player 2)
    F2                operator (test) menu: settings, free play
    9                 service coin (a free credit)
    Alt+Enter, F11    full screen
    F12               screenshot (frame.bmp)
    Escape            quit

Game controllers work too (the first is player 1, the second player 2): D-pad or left stick to move, X Y RB
punches, A B RT kicks, Start, and Back for a coin.

In the operator menu (F2), CONFIGURATION has the game and coin settings, including free play.

You can also start it as sfn.exe <path to sfiiin.zip>.

Street Fighter and Capcom are trademarks of Capcom. This project is not affiliated with or endorsed by Capcom.
