# Overview

This file contains my custom formats for Radarr and Sonarr. These formats identify specific resolutions, movie and show editions, video codecs, HDR standards, and release language indicators.

They help classify releases, apply custom format scores in quality profiles, and optionally preserve relevant format information when renaming files.

---
# 1. Resolution

These formats identify releases by their video resolution.

## HD - 1080p

```json
{
  "name": "HD - 1080p",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "HD - 1080p",
      "implementation": "ResolutionSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": 1080
      }
    }
  ]
}
```

## UHD - 2160p

```json
{
  "name": "UHD - 2160p",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "UHD - 2160p",
      "implementation": "ResolutionSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": 2160
      }
    }
  ]
}
```

---
# 2. Editions

Identifies releases that may contain alternate cuts or additional content.

## Director's Cut / Extended Edition

```json
{
  "name": "Director's Cut / Extended Edition",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "DirectorExtended",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(Director.?s.?Cut|DC|Extended(\\s*Cut)?|Special.?Edition|Unrated)"
      }
    }
  ]
}
```

---
# 3. Video Codecs

Identifies the video encoding format used by a release.

## AVC / x264

```json
{
  "name": "AVC / x264",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "x264Codec",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(x264|h264|avc)"
      }
    }
  ]
}
```

## HEVC / x265

```json
{
  "name": "HEVC / x265",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "x265Codec",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(x265|h265|hevc)"
      }
    }
  ]
}
```

---
# 4. HDR & Display Formats

Identifies HDR standards and specialized presentation formats.

## Dolby Vision

```json
{
  "name": "Dolby Vision",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "DolbyVision",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(\\bDV\\b|DoVi|Dolby.?Vision)"
      }
    }
  ]
}
```

## HDR

```json
{
  "name": "HDR",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "HDR",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(\\bHDR\\b)"
      }
    }
  ]
}
```

## HDR10

```json
{
  "name": "HDR10",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "HDR10",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(\\bHDR10\\b)"
      }
    }
  ]
}
```

## HDR10+

```json
{
  "name": "HDR10+",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "HDR10Plus",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(HDR10\\+|HDR10PLUS|HDR10-PLUS)"
      }
    }
  ]
}
```

## IMAX Enhanced

```json
{
  "name": "IMAX Enhanced",
  "includeCustomFormatWhenRenaming": true,
  "specifications": [
    {
      "name": "IMAX",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(IMAX|IMAX\\s*Enhanced)"
      }
    }
  ]
}
```

---
# 5. Language & Regional Releases

Identifies releases associated with a particular language or regional indicator.

## Italian Releases

```json
{
  "name": "Italian Releases",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "No Italian Releases",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": {
        "value": "(?i)(?:^|[^a-z])ita(?:[^a-z]|$)"
      }
    }
  ]
}
```

---

_End of document._
