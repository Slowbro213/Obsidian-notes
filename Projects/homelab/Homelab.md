This is my Kubernetes homelab project! I had some old laptops lying around at home, and I decided: why not put them to use? Thus the homelab was born! So far it consists of 2 laptops (will be expanded in the future), connected via my home WiFi. The 2 laptops are:

CachyOS (Control Node):
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="application/xml+xhtml; charset=UTF-8"/>
<title>stdin</title>
</head>
<body>
<pre>
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">          ▗▄▄▄       </span><span style="font-weight:bold;color:teal;">▗▄▄▄▄    ▄▄▄▖
</span><span style="font-weight:bold;color:blue;">          ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙  ▟███▛
</span><span style="font-weight:bold;color:blue;">           ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙▟███▛
</span><span style="font-weight:bold;color:blue;">            ▜███▙       </span><span style="font-weight:bold;color:teal;">▜██████▛
</span><span style="font-weight:bold;color:blue;">     ▟█████████████████▙ </span><span style="font-weight:bold;color:teal;">▜████▛     </span><span style="font-weight:bold;color:blue;">▟▙
    ▟███████████████████▙ </span><span style="font-weight:bold;color:teal;">▜███▙    </span><span style="font-weight:bold;color:blue;">▟██▙
</span><span style="font-weight:bold;color:teal;">           ▄▄▄▄▖           ▜███▙  </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">          ▟███▛             ▜██▛ </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">         ▟███▛               ▜▛ </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">▟███████████▛                  </span><span style="font-weight:bold;color:blue;">▟██████████▙
</span><span style="font-weight:bold;color:teal;">▜██████████▛                  </span><span style="font-weight:bold;color:blue;">▟███████████▛
</span><span style="font-weight:bold;color:teal;">      ▟███▛ </span><span style="font-weight:bold;color:blue;">▟▙               ▟███▛
</span><span style="font-weight:bold;color:teal;">     ▟███▛ </span><span style="font-weight:bold;color:blue;">▟██▙             ▟███▛
</span><span style="font-weight:bold;color:teal;">    ▟███▛  </span><span style="font-weight:bold;color:blue;">▜███▙           ▝▀▀▀▀
</span><span style="font-weight:bold;color:teal;">    ▜██▛    </span><span style="font-weight:bold;color:blue;">▜███▙ </span><span style="font-weight:bold;color:teal;">▜██████████████████▛
     ▜▛     </span><span style="font-weight:bold;color:blue;">▟████▙ </span><span style="font-weight:bold;color:teal;">▜████████████████▛
</span><span style="font-weight:bold;color:blue;">           ▟██████▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">          ▟███▛▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">         ▟███▛  ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">         ▝▀▀▀    ▀▀▀▀▘       </span><span style="font-weight:bold;color:teal;">▀▀▀▘</span><span style="font-weight:bold;"></span>

<span style="font-weight:bold;color:blue;">slowking</span>@<span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">cachyos</span>
----------------
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">OS</span>: NixOS 25.11 (Xantusia) x86_64
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Host</span>: 82VG (IdeaPad 1 15AMN7)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Kernel</span>: Linux 6.12.93
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Uptime</span>: 1 hour, 58 mins
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Packages</span>: 451 (nix-system)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Shell</span>: bash 5.3.3
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Display (BOE0AED)</span>: 1920x1080 in 16&quot;, 60 Hz [Built-in]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Terminal</span>: /dev/pts/1
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">CPU</span>: AMD Ryzen 3 7320U (8) @ 4.15 GHz
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">GPU</span>: AMD Radeon 610M [Integrated]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Memory</span>: 8.57 GiB / 13.39 GiB (<span style="filter: contrast(70%) brightness(190%);color:olive;">64%</span>)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Swap</span>: 13.50 MiB / 6.69 GiB (<span style="color:green;">0%</span>)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Disk (/)</span>: 103.95 GiB / 237.86 GiB (<span style="color:green;">44%</span>) - xfs
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Local IP (wlan0)</span>: 192.168.1.31/24
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Battery (L21D3PF0)</span>: <span style="color:green;">99%</span> [Charging, AC Connected]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Locale</span>: en_US.UTF-8

<span style="background-color:black;">   </span><span style="background-color:red;">   </span><span style="background-color:green;">   </span><span style="background-color:olive;">   </span><span style="background-color:blue;">   </span><span style="background-color:purple;">   </span><span style="background-color:teal;">   </span><span style="background-color:gray;">   </span>
<span style="filter: contrast(70%) brightness(190%);background-color:black;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:red;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:green;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:olive;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:blue;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:purple;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:teal;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:gray;">   </span>
</pre>
</body>
</html>

Tux (Worker Node):
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="application/xml+xhtml; charset=UTF-8"/>
<title>stdin</title>
</head>
<body>
<pre>
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">          ▗▄▄▄       </span><span style="font-weight:bold;color:teal;">▗▄▄▄▄    ▄▄▄▖
</span><span style="font-weight:bold;color:blue;">          ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙  ▟███▛
</span><span style="font-weight:bold;color:blue;">           ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙▟███▛
</span><span style="font-weight:bold;color:blue;">            ▜███▙       </span><span style="font-weight:bold;color:teal;">▜██████▛
</span><span style="font-weight:bold;color:blue;">     ▟█████████████████▙ </span><span style="font-weight:bold;color:teal;">▜████▛     </span><span style="font-weight:bold;color:blue;">▟▙
    ▟███████████████████▙ </span><span style="font-weight:bold;color:teal;">▜███▙    </span><span style="font-weight:bold;color:blue;">▟██▙
</span><span style="font-weight:bold;color:teal;">           ▄▄▄▄▖           ▜███▙  </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">          ▟███▛             ▜██▛ </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">         ▟███▛               ▜▛ </span><span style="font-weight:bold;color:blue;">▟███▛
</span><span style="font-weight:bold;color:teal;">▟███████████▛                  </span><span style="font-weight:bold;color:blue;">▟██████████▙
</span><span style="font-weight:bold;color:teal;">▜██████████▛                  </span><span style="font-weight:bold;color:blue;">▟███████████▛
</span><span style="font-weight:bold;color:teal;">      ▟███▛ </span><span style="font-weight:bold;color:blue;">▟▙               ▟███▛
</span><span style="font-weight:bold;color:teal;">     ▟███▛ </span><span style="font-weight:bold;color:blue;">▟██▙             ▟███▛
</span><span style="font-weight:bold;color:teal;">    ▟███▛  </span><span style="font-weight:bold;color:blue;">▜███▙           ▝▀▀▀▀
</span><span style="font-weight:bold;color:teal;">    ▜██▛    </span><span style="font-weight:bold;color:blue;">▜███▙ </span><span style="font-weight:bold;color:teal;">▜██████████████████▛
     ▜▛     </span><span style="font-weight:bold;color:blue;">▟████▙ </span><span style="font-weight:bold;color:teal;">▜████████████████▛
</span><span style="font-weight:bold;color:blue;">           ▟██████▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">          ▟███▛▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">         ▟███▛  ▜███▙       </span><span style="font-weight:bold;color:teal;">▜███▙
</span><span style="font-weight:bold;color:blue;">         ▝▀▀▀    ▀▀▀▀▘       </span><span style="font-weight:bold;color:teal;">▀▀▀▘</span><span style="font-weight:bold;"></span>

<span style="font-weight:bold;color:blue;">slowking</span>@<span style="font-weight:bold;"></span><span style="font-weight:bold;color:blue;">tux</span>
------------
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">OS</span>: NixOS 25.11 (Xantusia) x86_64
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Host</span>: Latitude E5470
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Kernel</span>: Linux 6.12.93
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Uptime</span>: 1 hour, 55 mins
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Packages</span>: 451 (nix-system)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Shell</span>: bash 5.3.3
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Display (BOE0653)</span>: 1920x1080 in 14&quot;, 60 Hz [Built-in]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Terminal</span>: /dev/pts/1
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">CPU</span>: Intel(R) Core(TM) i5-6200U (4) @ 2.80 GHz
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">GPU</span>: Intel HD Graphics 520 @ 1.00 GHz [Integrated]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Memory</span>: 3.06 GiB / 7.65 GiB (<span style="color:green;">40%</span>)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Swap</span>: 0 B / 3.83 GiB (<span style="color:green;">0%</span>)
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Disk (/)</span>: 100.33 GiB / 237.86 GiB (<span style="color:green;">42%</span>) - xfs
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Local IP (wlp1s0)</span>: 192.168.1.25/24
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Battery (DELL J8FXW74)</span>: <span style="filter: contrast(70%) brightness(190%);color:red;">6%</span> [AC Connected]
<span style="font-weight:bold;"></span><span style="font-weight:bold;color:teal;">Locale</span>: en_US.UTF-8

<span style="background-color:black;">   </span><span style="background-color:red;">   </span><span style="background-color:green;">   </span><span style="background-color:olive;">   </span><span style="background-color:blue;">   </span><span style="background-color:purple;">   </span><span style="background-color:teal;">   </span><span style="background-color:gray;">   </span>
<span style="filter: contrast(70%) brightness(190%);background-color:black;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:red;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:green;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:olive;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:blue;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:purple;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:teal;">   </span><span style="filter: contrast(70%) brightness(190%);background-color:gray;">   </span>
</pre>
</body>
</html>

The reason behind the names of the nodes is due to what they used to be. These two laptops used to run the [CachyOS](https://cachyos.org/) and [Gentoo](https://www.gentoo.org/) Linux distros, respectively, but have since moved to the [NixOS](https://nixos.org/) Linux distro due to the control NixOS provides with its declarative capabilities. The [Tux](https://en.wikipedia.org/wiki/Tux_(mascot)) name, though, just happened to appear on its own after I installed NixOS on the ex-Gentoo node.

I mainly use this cluster to learn Kubernetes and general DevOps concepts such as Linux administration, CI/CD pipelines, etc., but also to self-host projects of mine. One of these projects, which I'll briefly mention, is a website I made for [Epoka's](https://epoka.edu.al/) programming club. It served as a platform for hosting programming competitions among students, with [LeetCode](https://leetcode.com/)-style exercises. The old, functioning website is no longer online; the website that can be found at https://epokaprogramming.com is a read-only version with a separate database, meant only to show what once was.

Here I'll be documenting everything about my homelab! From physical hardware management to deploying full-fledged applications online!

---

Next: [[Project Layout]] →