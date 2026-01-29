---
title: windows desktop right click dialog display err
---

https://www.gamersky.com/handbook/201608/797053.shtml


taskmgr

regedit

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\FlightedFeatures

新建DWORD（32位）值，重命名为ImmersiveContextMenu，数值数据直接使用默认的“0”
