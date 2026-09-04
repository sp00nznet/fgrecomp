# Triage: `libfg.so`

- machine: `EM_AARCH64`
- file: 45.0 MB, `.text`: 28.6 MB @ `0x5585f0`
- instructions: 7,144,062 (272 distinct mnemonics)
- functions from `.eh_frame`: **85,702** covering 100.0% of `.text`
- undisassembled bytes: 2

## Shim surface

Linked libraries -- every one of these is a shim you write:

- `libGLESv2.so`
- `liblog.so`
- `libandroid.so`
- `libOpenSLES.so`
- `libEGL.so`
- `libz.so`
- `libGLESv1_CM.so`
- `libdl.so`
- `libc.so`
- `libm.so`

Undefined symbols to satisfy: **386**  
Exported symbols: **11994**

JNI entry points (the Java glue you must replace): **92**

```
Java_com_sgn_gs_SGNMobile_dispatchPlatformEvent
Java_com_tinyco_familyguy_FGBaseGameActivity_onSettingsClosed
Java_com_tinyco_familyguy_PlatformAppOnboard_onPresentationBegan
Java_com_tinyco_familyguy_PlatformAppOnboard_onPresentationEnded
Java_com_tinyco_familyguy_PlatformIncentVideo_isShowingOffers
Java_com_tinyco_familyguy_PlatformIncentVideo_onRequestBrandEngageOffers
Java_com_tinyco_familyguy_PlatformIncentVideo_onShowBrandEngageOffers
Java_com_tinyco_familyguy_PlatformIronSource_getProdUrl
Java_com_tinyco_familyguy_PlatformIronSource_getStageUrl
Java_com_tinyco_familyguy_PlatformIronSource_onAvailabilityChanged
Java_com_tinyco_familyguy_PlatformIronSource_onReceivedOfferwallCredits
Java_com_tinyco_familyguy_PlatformIronSource_onRewardedVideoAdClosedHandler
Java_com_tinyco_familyguy_PlatformIronSource_setPlayBackgroundMusic
Java_com_tinyco_familyguy_PlatformShare_sendShareToFacebook
Java_com_tinyco_familyguy_PlatformShare_shareComplete
Java_com_tinyco_familyguy_PlatformTapResearch_getAppId
Java_com_tinyco_familyguy_PlatformTapResearch_getPlacementId
Java_com_tinyco_familyguy_PlatformTapResearch_onTapResearchPlacementSetupComplete
Java_com_tinyco_familyguy_PlatformTapResearch_onTapResearchReceivedReward
Java_com_tinyco_familyguy_PlatformTapResearch_onTapResearchSurveyWallClosed
Java_com_tinyco_familyguy_PlatformTapjoy_getPlacementName
Java_com_tinyco_familyguy_PlatformTapjoy_onOfferwallClosed
Java_com_tinyco_familyguy_PlatformTapjoy_onOfferwallOpened
Java_com_tinyco_familyguy_VideoViewActivity_notifyDone
Java_com_tinyco_familyguy_VideoViewActivity_notifyError
Java_com_tinyco_familyguy_VideoViewActivity_notifySkip
Java_com_tinyco_griffin_PlatformFacebook_handleDialogClosed
Java_com_tinyco_griffin_PlatformFacebook_handleDialogClosedWithFbIds
Java_com_tinyco_griffin_PlatformFacebook_handleLocalResponse
Java_com_tinyco_griffin_PlatformFacebook_handleResponse
Java_com_tinyco_griffin_PlatformGoogle_handleLocalResponse
Java_com_tinyco_griffin_PlatformGoogle_handleResponse
Java_com_tinyco_griffin_PlatformGoogle_setAchievementsLoadedFlag
Java_com_tinyco_griffin_PlatformUtils_00024DownloaderRequest_doCallback
Java_com_tinyco_griffin_PlatformUtils_00024WebRequest_doCallback
Java_com_tinyco_griffin_PlatformUtils_createSentryMessage
Java_com_tinyco_griffin_PlatformUtils_doNativeActionWithParameter
Java_com_tinyco_griffin_PlatformUtils_isAmazonBuild
Java_com_tinyco_griffin_PlatformUtils_isBackgroundMusicPlaying
Java_com_tinyco_griffin_PlatformUtils_isGoogleBuild
...
```

## Constructs the lifter must special-case

| construct | count |
|---|---|
| indirect-branch | 134,678 |
| exclusive | 97,798 |
| lse-atomic | 0 |
| barrier | 363 |
| syscall | 0 |
| sysreg | 13,095 |
| pointer-auth | 0 |

## Top mnemonics

| mnemonic | count |
|---|---|
| `ldr` | 1,080,457 |
| `mov` | 893,758 |
| `add` | 780,747 |
| `bl` | 498,198 |
| `str` | 472,973 |
| `cmp` | 323,115 |
| `b` | 287,409 |
| `stp` | 225,318 |
| `cbz` | 217,660 |
| `adrp` | 208,451 |
| `ldp` | 205,682 |
| `ldrb` | 173,687 |
| `cbnz` | 143,374 |
| `sub` | 132,140 |
| `blr` | 123,133 |
| `b.eq` | 120,354 |
| `b.ne` | 118,630 |
| `tbz` | 94,224 |
| `strb` | 77,823 |
| `tbnz` | 75,564 |
| `stur` | 72,267 |
| `ldur` | 67,259 |
| `ret` | 66,473 |
| `csel` | 49,448 |
| `lsr` | 33,894 |
