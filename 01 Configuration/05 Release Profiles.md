# Overview

This document outlines the Radarr and Sonarr quality profiles for standard HD (1080p) and UHD (2160p) movies/TV. Each profile defines preferred release qualities, upgrade behavior, and custom format scores to prioritize specific editions, HDR formats, codecs, and IMAX Enhanced releases.

---
# 1. General Profile

## ANY

```
Name: ANY

Upgrades Allowed: No

Minimum Custom Format Score: 0

Language: Original

Qualities (In Order):
1. Bluray-2160p
2. WEB-2160p
3. Remux-2160p
4. HDTV-2160p
5. Bluray-1080p
6. WEB-1080p
7. Remux-1080p
8. HDTV-1080p

Custom Format Scores:
UHD - 2160p:                        25
HD - 1080p:                         20
Director's Cut / Extended Version:  18
IMAX Enhanced:                      17
Dolby Vision:                       15
HDR10+:                             10
HEVC / x265:                        8
HDR10:                              6
HDR:                                3
AVC / x264:                         0
Italian Releases                    -10
```

---
# 2. HD Profiles (1080p)

## 2.1 HD-1080p | Director's Cut

Prioritizes extended editions and director's cuts.

```
Name: HD-1080p-Director's Cut

Upgrades Allowed: Yes
Upgrade Until: Bluray-1080p

Minimum Custom Format Score: 8
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-1080p
2. WEB-1080p
3. Remux-1080p
4. HDTV-1080p

Custom Format Scores:
Director's Cut / Extended Version:  10
HEVC / x265:                        6
AVC / x264:                         3
UHD - 2160p:                        0
HD - 1080p:                         0
IMAX Enhanced:                      0
Dolby Vision:                       0
HDR10+:                             0
HDR10:                              0
HDR:                                0
Italian Releases                    -10
```

## 2.2 HD-1080p | IMAX

Prioritizes IMAX Enhanced releases.

```
Name: HD-1080p-IMAX

Upgrades Allowed: Yes
Upgrade Until: Bluray-1080p

Minimum Custom Format Score: 8
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-1080p
2. WEB-1080p
3. Remux-1080p
4. HDTV-1080p

Custom Format Scores:
IMAX Enhanced:                      10
UHD - 2160p:                        0
HD - 1080p:                         0
Director's Cut / Extended Version:  0
Dolby Vision:                       0
HDR10+:                             0
HEVC / x265:                        0
HDR10:                              0
HDR:                                0
AVC / x264:                         0
Italian Releases                    -10
```

## 2.3 HD-1080p | Optimized

Combines edition, HDR, codec, and IMAX preferences in one profile.

```
Name: HD-1080p-Optimized

Upgrades Allowed: Yes
Upgrade Until: Bluray-1080p

Minimum Custom Format Score: 0
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-1080p
2. WEB-1080p
3. Remux-1080p
4. HDTV-1080p

Custom Format Scores:
HD - 1080p:                         20
Director's Cut / Extended Version:  18
IMAX Enhanced:                      17
Dolby Vision:                       15
HDR10+:                             10
HEVC / x265:                        8
HDR10:                              6
HDR:                                3
AVC / x264:                         0
UHD - 2160p:                        -10
Italian Releases                    -100
```

---
# 3. UHD Profiles (2160p)

## 3.1 UHD-2160p | Director's Cut

Prioritizes extended editions and director's cuts at 2160p.

```
Name: UHD-2160p-Director's Cut

Upgrades Allowed: Yes
Upgrade Until: Bluray-2160p

Minimum Custom Format Score: 8
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-2160p
2. WEB-2160p
3. Remux-2160
4. HDTV-2160p

Custom Format Scores:
Director's Cut / Extended Version:  10
HEVC / x265:                        5
UHD - 2160p:                        0
HD - 1080p:                         0
IMAX Enhanced:                      0
Dolby Vision:                       0
HDR10+:                             0
HDR10:                              0
HDR:                                0
AVC / x264:                         -20
Italian Releases                    -30
```

## 3.2 UHD-2160p | IMAX

Prioritizes IMAX Enhanced releases at 2160p.

```
Name: UHD-2160p-IMAX

Upgrades Allowed: Yes
Upgrade Until: Bluray-2160p

Minimum Custom Format Score: 8
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-2160p
2. WEB-2160p
3. Remux-2160p
4. HDTV-2160p

Custom Format Scores:
IMAX Enhanced:                      10
HEVC / x265:                        5
Director's Cut / Extended Version:  0
UHD - 2160p:                        0
HD - 1080p:                         0
Dolby Vision:                       0
HDR10+:                             0
HDR10:                              0
HDR:                                0
AVC / x264:                         -20
Italian Releases                    -30
```

## 3.3 UHD-2160p | Optimized

Combines edition, HDR, codec, Dolby Vision, and IMAX preferences in one profile.

```
Name: UHD-2160p-Optimized

Upgrades Allowed: Yes
Upgrade Until: Bluray-2160p

Minimum Custom Format Score: 0
Upgrade Until Custom Format Score: 0
Minimum Custom Format Score Increment: 1

Language: Original

Qualities (In Order):
1. Bluray-2160p
2. WEB-2160p
3. Remux-2160p
4. HDTV-2160p

Custom Format Scores:
UHD - 2160p:                        20
Director's Cut / Extended Version:  18
IMAX Enhanced:                      17
Dolby Vision:                       15
HDR10+:                             10
HEVC / x265:                        8
HDR10:                              6
HDR:                                3
HD - 1080p:                         -10
AVC / x264:                         -20
Italian Releases                    -100
```

---

_End of document._
