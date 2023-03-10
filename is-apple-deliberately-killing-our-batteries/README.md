<!--
Title: Is Apple deliberately killing our batteries?
Description: Exploring if Apple is deliberately killing our batteries… and what one can do about it.
Cover image: macbook-air.jpg
Publication date: 2023-03-10T12:14:47.264Z
Listed: true
-->

<span class="drop-cap">I</span>f you are reading this story, you likely have an iPhone or a Mac that you would like to use for as long as possible.

Unfortunately, both iPhone and Mac devices have batteries that are not user serviceable even though they are [consumables](https://support.apple.com/en-us/HT208387) and have limited lifespan.

According to [research](https://batteryuniversity.com/article/bu-808-how-to-prolong-lithium-based-batteries), lithium batteries should ideally be charged to a maximum of 80% of their capacity and discharged to a minimum of 20% to increase their lifespan.

> Electric car batteries should not, generally, be charged to 100%. Long-term, this reduces the battery's longevity, and Tesla cars actually charge up to 90% by default. [Mashable](https://mashable.com/article/tesla-battery-charge-max)

> So really, if you were super-keen on keeping your battery living as long as possible, you should keep its charge between 20 and 80 per cent. [Wired](https://www.wired.co.uk/article/how-to-improve-battery-life-tips-myths-smartphones)

As a result, Apple has implemented a feature called [Optimized Battery Charging](https://support.apple.com/en-us/HT210512) available and enabled by default starting on iOS 13 and macOS Big Sur (requires “Location Services” which isn’t great for privacy).

When Apple Silicon MacBook Air or Pro is chronically plugged (when using a dock for example), Optimized Battery Charging is supposed to [pause](https://support.apple.com/en-us/HT212049) charging when battery is charged to 80% to preserve its lifespan.

Problem is… **it doesn’t**. At least this is what I noticed on my M1 MacBook Air (corroborated on my girlfriend’s M1 MacBook Air) when I investigated why capacity of battery was at 89% after only 50 charging cycles.

> Heads-up: to contribute to project, please share cycle count, maximum capacity and purchase date of your Mac in the comments of YouTube episode.
>
> To find cycle count and maximum capacity, run following command (without the `$`) in “Terminal” app.
>
> ```console
> $ system_profiler SPPowerDataType | grep -E "Cycle Count|Maximum Capacity" | awk '{printf "%s ",$3}'
> 50 89%
> ```
>
> To find purchase date, please go to [https://checkcoverage.apple.com/](https://checkcoverage.apple.com/), enter serial number and hit “Submit”.

Another common (and very expensive) issue with chronically plugged Macs is [swollen batteries](https://www.ifixit.com/Wiki/What_to_do_with_a_swollen_battery). To be fair, this is not an Apple-specific issue but rather one related to Lithium batteries in general. That said, I believe Apple could easily do something about it such as asking users to discharge battery to 20% once in a while.

When I say **easily**, I mean it… here is how one can discharge battery of Apple Silicon Macs to 20% while plugged (using a secret trick).

## Caveats

- When copy/pasting commands that start with `$`, strip out `$` as this character is not part of the command
- When copy/pasting commands that start with `cat << "EOF"`, select all lines at once (from `cat << "EOF"` to `EOF` inclusively) as they are part of the same (single) command

## Setup guide

> Shout-out to [actuallymentor](https://github.com/actuallymentor/battery) for opening my eyes **(use at your own risk, may void warranty)**

### Step 1: install Xcode Command Line Tools

```console
$ xcode-select --install
```

### Step 2: clone [smcFanControl](https://github.com/hholtmann/smcFanControl) GitHub repository

```console
$ git clone https://github.com/hholtmann/smcFanControl
```

### Step 3: compile and install [smc](https://github.com/hholtmann/smcFanControl/tree/master/smc-command) command

```console
$ cd smcFanControl/smc-command

$ make

$ sudo cp smc /usr/local/bin/
```

### Step 4: create `discharge.sh` convenience script

```console
$ cat << "EOF" | sudo tee /usr/local/bin/discharge.sh
#! /bin/sh

function cancel()
{
  printf "\n"
  sudo smc -k CH0I -w 00
  exit 0
}

trap cancel INT

bold=$(tput bold)
red=$(tput setaf 1)
normal=$(tput sgr0)

target=20

sudo smc -k CH0I -w 01

printf "$bold$red%s$normal\n" "Discharging battery to $target% (press ctrl+c to cancel)"

while [ $(pmset -g batt | grep --extended-regexp --only-matching "\d+%" | cut -d% -f1) -gt $target ]; do
  sleep 60
done

cancel
EOF
```

### Step 5: make `discharge.sh` executable

```console
$ sudo chmod +x /usr/local/bin/discharge.sh
```

## Usage guide

```console
$ discharge.sh
Discharging battery to 20% (press ctrl+c to cancel)
```
