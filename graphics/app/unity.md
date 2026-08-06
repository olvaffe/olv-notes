# Unity

## Overview

- engine history
  - 2005, Unity 1.0
  - 2007, Unity 2.0
  - 2010, Unity 3.0
  - 2012, Unity 4.0
  - 2015, Unity 5
  - 2017, Unity 2017
  - 2018, Unity 2018
  - 2019, Unity 2019
    - 2019.3 introduced URP (universal render pipeline)
  - 2020, Unity 2020
  - 2021, Unity 2021
  - 2022, Unity 2022
  - 2023, Unity 2023
  - 2024, Unity 6
    - URP becomes default

## Demo: Boat Attack

- <https://github.com/Unity-Technologies/BoatAttack>
- Unity 2019.3
- adb
  - `adb shell am start --windowingMode 1 -n com.UnityTechnologies.BoatAttack/com.unity3d.player.UnityPlayerActivity`
    - optional `-e unity \"-quality:hq\"`
  - `adb shell am force-stop com.UnityTechnologies.BoatAttack`

## Demo: Viking Village

- <https://assetstore.unity.com/packages/essentials/tutorial-projects/viking-village-urp-29140>
- Unity 2020.2 (originally Unity 5)
- adb
  - `adb shell am start -n com.UnityTechnologies.VikingVillage/com.unity3d.player.UnityPlayerActivity`
