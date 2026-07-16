## Overview
As streaming continues to grow, innovation around CTV ad formats does too. In 2024, the market wanted standards. Member demand drove Tech Lab to launch the Ad Format Hero initiative, which garnered hundreds of submissions from dozens of companies. Our Taskforce sorted, categorized and found the commonalities in the submissions to create a new portfolio of CTV ads that covered the majority of use cases. 

In order to scale these newly defined formats, we needed a standard way to signal them in the bid request and to provide the assets.

The working group sought balance between two main goals: First, to ensure that buyers knew what they were buying. No one should be able to accidentally purchase a pause ad. Buyers should opt in because they want to buy these new formats. Second, where possible, we wanted to fit into existing workflows, so long as the first goal was still met. 

This is how we coalesced around the video object in the request and the VAST response for the majority of use cases. We have one notable exception for menu ads. In all of the examples we received, menu ads were treated like traditional native ads (e.g. the operating system received assets and assembled the creative). Instead of adding that type of functionality to VAST, we leveraged the existing Native API specification.

## How to Use the Signals
This section describes how the new AdCOM enumerations, Native API placement types, and VAST non-linear updates work together to signal and deliver the six new CTV ad formats defined by the Ad Format Hero project: Pause, Screensaver, Overlay, Squeezeback, In-Scene, and Menu.

These signals are intended to be implementation-agnostic. The same request and response patterns apply whether the winning creative is rendered by a client-side player or by a server-side stitcher implementation. 

A few cross-cutting notes before the format-by-format guidance:
- The five non-linear formats (Pause, Screensaver, Overlay, Squeezeback, In-Scene) reuse the existing OpenRTB Video imp object. No new top-level RTB attributes are introduced; differentiation comes entirely from expanded AdCOM enumerations on plcmt, pos, and playbackmethod, paired with the updated VAST <NonLinearAds> node.
- The Menu format uses the OpenRTB Native imp object. Existing Native placement types (1–4) cover most CTV menu/tile contexts, and the existing 500+ exchange-defined range remains available for exchange-specific placements where 1–4 do not fit.
- VAST <Extensions> are used to round-trip pos and attr so that publishers, stitchers, and measurement vendors can validate the creative experience independently of the bid response payload. This is particularly important when the VAST is consumed downstream of the original RTB transaction (for example, by a stitcher that does not see the bid object).

### Overall Flow

The two paths outlined below illustrate the end-to-end signaling flow. The first covers the five non-linear CTV formats; the second covers Menu Ads.

In both flows, the bid request is the authoritative description of the opportunity. The bid response should return a creative that matches the requested format, position, playback context, and supported creative experience, and the receiving player or stitcher should render the winning creative and fire the associated tracking URLs.

### Signaling the Five Non-Linear CTV Formats

For Pause, Screensaver, Overlay, Squeezeback, and In-Scene, the exchange contract is:

- <b>Bid request</b> — Expressed using the OpenRTB Video imp object with the expanded AdCOM enumerations for plcmt (placement subtype, identifying the format), pos (placement position, identifying the on-screen treatment), and playbackmethod (identifying how the ad is initiated). Refer to the Format-to-Signal Reference Table for the specific value combinations per format. Publishers may also use battr to block creative attributes that their placement does not support (see Declaring Creative Experience with battr and attr).
- <b>Bid response</b> — Includes ad creatives matching the requested format, with adm containing the updated VAST non-linear payload as specified in the VAST Response Update section. The <NonLinearAds> node now supports <MediaFiles> for both image and video creative files, enabling the same creative-delivery model as <LinearAds>. The DSP echoes the format context into VAST <Extensions> (pos, attr) so that downstream consumers can validate the experience, and uses attr on the bid object to declare creative motion characteristics.
- <b>Player or stitcher consumption</b> — The player or stitcher consumes the non-linear VAST of the winning bid to render the ad in the appropriate placement context (paused content, idle screen, overlay region, squeezeback frame, or in-scene insertion). It fires the tracking pixels declared in the VAST, including creativeView, overlayViewDuration, and quartile events when <Duration> is present. See Handling Duration for guidance on when <Duration> is required.

For complete request/response/VAST examples, refer to the Examples section.

### Signaling Menu / Tile Ads

Menu and tile-screen ads have distinct UX requirements that align well with the asset-separation model of Native, so they are transacted through the OpenRTB Native imp object.

- <b>Bid request</b> — Expressed using the Native API with the appropriate plcmttype value 1 should be used for tile slots on the menu, and 3 should be used for headline banners. Both of these plcmttype exist outside of the CTV context, so it is assumed the bid request will include appropriate device information to indicate CTV inventory: device.devicetype = 3 is desirable to include at a minimum. The publisher signals supported creative types by including the corresponding asset objects: an img asset for image-only placements, a video asset for video-only placements, or both for mixed placements that accept either.
- <b>Bid response</b> — Includes the supporting creative type(s) requested, delivered either as direct creative URLs (for image assets) or as a VAST tag embedded in the vasttag field of a video asset. The Native API already supports embedded VAST in the Native response, so no new mechanism is required.
- <b>CTV platform consumption</b> — The platform renders the image or video creative of the winning bid in the menu/tile placement and fires the tracking pixels declared in the Native bid response (eventtrackers) and, for video creatives, in the embedded VAST.

For complete request/response examples covering video-only, image-only, and mixed Menu placements, refer to the Menu Ad examples.

### Format-to-Signal Reference Table 

The following table summarizes the AdCOM enumeration values used to signal each new CTV ad format as defined in the <a href="https://github.com/InteractiveAdvertisingBureau/Ad-Format-Guidelines-for-Digital-Video-CTV/blob/main/v2.0.md">CTV Ad Format Guidelines</a>. Where multiple values are listed, any one applies; the choice depends on the specific on-screen treatment.

| Format | Request object | Format signal | Position signal | Playback signal | Expected response pattern |
|---|---|---|---|---|---|
| Pause | imp.video | plcmt = 5 | pos = 7 fullscreen, 8 partial screen | 8 sound on, 9 sound off |bid.adm with NonLinear VAST; static image or video/cinemagraph asset; bid.attr declares actual experience |
| Screensaver | imp.video | plcmt = 6 | pos = 7 fullscreen, 8 partial screen | 10 sound on, 11 sound off | bid.adm with NonLinear VAST; often static, but may be motion-based if supported |
| Overlay | imp.video | plcmt = 7 |pos = 9, 10, 14, 15, or 5 depending geometry | Typically 1 or 2, based on whether the overlay begins with sound on or off; example uses 2 | bid.adm with NonLinear VAST; image or video asset; include Duration when time-based tracking is expected |
| Squeezeback | imp.video | plcmt = 8 | pos = 11, 12, or 13,16,17 depending layout | Typically 1 or 2; example uses 2 | bid.adm with NonLinear VAST; image or video asset; include Duration when time-based tracking is expected |
| In-scene | imp.video | plcmt = 9 | NA | Typically 1 or 2, depending whether the inserted creative experience autostarts with sound on or off | bid.adm with NonLinear VAST; static or video insert; bid.attr should still describe the actual motion state |


| Menu execution | Request object | Placement signal | Asset declaration | Expected response pattern |
|---|---|---|---|---|
| Tile / in-menu unit | imp.native.request.native | plcmttype = 1 when feed/tile semantics fit | assets[] specifies image, video, or both | Native response with img.url, video.vasttag, or both |
| headline banner | imp.native.request.native | plcmttype = 3 when outside-core-content/banner semantics fit | assets[] specifies image, video, or both | Native response with direct asset URL and/or embedded VAST |
| Exchange-specific menu subtype | imp.native.request.native | plcmttype = 500+ or documented ext | assets[] still declares image, video, or both | Same Native response pattern; exchange-defined subtype only adds menu-specific semantics |


### Declaring Creative Experience with battr and attr

Publisher capabilities vary in ways that the standard MIME-type and protocol signaling cannot fully express. In particular, a publisher may technically support delivery of a video/mp4 file but expect a static visual experience in the placement — for example, an Overlay placement that renders an MP4 frame as a still image, or a Pause placement designed for cinemagraph-style limited motion rather than full-motion video.

To express this constraint, the proposal extends the AdCOM Creative Attributes list with three new motion-related values:

- <b>21 — Static Visual.</b> Creative renders as a static visual with no perceptible motion, even when delivered as a video file.
- <b>22 — Limited Motion (Cinemagraph).</b> Creative contains subtle or localized motion within an otherwise static composition.
- <b>23 — Full-Motion Video.</b> Creative contains continuous, scene-level motion typical of standard video.

Publishers should use battr in the bid request to block creative attributes their placement does not support. For example, a publisher offering an Pause Ad placement that only supports static experiences should set battr: [22, 23] to block both Limited Motion and Full-Motion Video, even when the placement accepts MP4 delivery. DSPs should consider setting attr on the bid response to declare the motion characteristic of the delivered creative, so that publishers can validate the experience matches the placement's intent.

This pattern allows the supply side to express experiential constraints separately from technical-format support, and gives the demand side a clear signal of what kind of creative will actually pass the publisher's filter.

### QR Code Signaling

Many of these CTV placements may include a QR code because they are concurrent, pause-state, or idle-state experiences. If the buyer is returning a creative that contains an advertiser QR code, it should declare that in attr. When the platform needs standardized information about the QR destination or placement geometry, the VAST CreativeExtensions block may be used to carry the QR code scan URL, optional rendered QR image URL, position, and size. 

## Handling Duration
The VAST specification already supports duration and should be used when the duration of the ad is known. 

Some instances where this might not be known when the VAST is sent are static image ads being used for pause, screensaver and overlays. In those cases, including the duration in VAST is optional, although an indication of duration should have been expressed by the publisher in the bid request.

## Handling “Custom” vs “New” Ad Units Expression
It is understood that the industry will need time to migrate to this new standard. Any translation from a custom expression of these formats to the new standard may be done by the technology partners, assuming they have the necessary information from the advertiser or publisher partner to make that determination.

## Deep Linking & Clickthrough to Destination 
If advertisers expect a traditional clickthrough with their ad, they should continue to use clickthrough url in VAST. Publisher execution of this feature may vary, and advertisers should check with their publisher partner to understand the capabilities.

## SIMID Based Interactivity Ad Units
All formats in the CTV Ad Portfolio may be interactive. SIMID is the industry standard for interactivity. In the request, feature support may be signaled using the api attribute in the video object. 

Necessary support may be signaled by the advertiser in VAST:

```xml
	LinearAds:
	use <InteractiveCreativeFile apiFramework=”SIMID”> to declare the creative markup in <MediaFile> is SIMID 


	NonLinearAds:
	use <IFrameResource apiFramework=”SIMID”> to both declare SIMID and provide the creative markup.
```

## Purpose of VAST ext & Failure Cases
Advertisers should respect the values provided in the bid request by the publisher and ensure that any creative sent is a match. 
Publishers integrations should be resilient when receiving attributes or assets that they didn’t expect. They should ensure they have a way to adhere to their preferred user experience regardless of the creative received. 


## Examples

Menu Ad - Video Only - OpenRTB Request
```json
{
    "id": "80ce30c5-3b42-4d58-9314-1ad1fe34ac12",
    "imp": [
      {
        "id": "1",
        "native": {
          "request": {
            "native": {
              "ver": "1.2",
              "context": 1,
              "contextsubtype": 12,
              "plcmttype": 3,
              "assets": [
                {
                  "id": 1,
                  "required": 1,
                  "video": {
                    "mimes": ["video/mp4"],
                    "minduration": 5,
                    "maxduration": 30,
                    "protocols": [3, 6, 7, 8, 11, 12, 13, 14, 15, 16],
                    "w": 1920,
                    "h": 1080,
                    "placement": 2,
                    "linearity": 1,
                    "skip": 0,
                    "playbackmethod": [2, 9],
                    "api": [8, 9]
                  }
                }
              ],
              "eventtrackers": [
                { "event": 1, "methods": [1, 2] },
                { "event": 2, "methods": [1] }
              ]
            }
          },
          "ver": "1.2"
        },
        "instl": 0,
        "tagid": "native-video-ad-unit-1",
        "bidfloor": 5.0,
        "bidfloorcur": "USD",
        "secure": 1,
        "exp": 9000
      }
    ],
    "app": {
      "id": "159384962",
      "name": "Example CTV App",
      "bundle": "com.example.ctvapp",
      "storeurl": "https://example.com/app",
      "cat": ["IAB1"],
      "publisher": {
        "id": "82524850",
        "name": "Example Publisher"
      }
    },
    "device": {
      "ua": "Mozilla/5.0 (Linux; SMART-TV 1.0) AppleWebKit/537.36 (KHTML, like Gecko) SMART-TV/4.0 TV Safari/537.36",
      "geo": {
        "country": "CAN",
        "region": "QC",
        "city": "Montreal"
      },
      "ip": "192.168.1.1",
      "devicetype": 3,
      "make": "Example TV",
      "model": "Smart TV",
      "os": "Example TV OS",
      "osv": "6.5",
      "w": 1920,
      "h": 1080,
      "ifa": "2d255aff-4389-418e-8f7b-3ff06d7e8901",
      "connectiontype": 1
    },
    "user": {
      "id": "user-12345"
    },
    "at": 2,
    "tmax": 250,
    "cur": ["USD"],
    "bcat": ["IAB25", "IAB26"],
    "badv": ["competitor.com"]
  }
```

Menu Ad - Video Only - OpenRTB Response

```json
{
    "id": "80ce30c5-3b42-4d58-9314-1ad1fe34ac12",
    "seatbid": [
      {
        "bid": [
          {
            "id": "bid-1",
            "impid": "1",
            "price": 7.50,
            "nurl": "https://example-dsp.com/win?price=${AUCTION_PRICE}&id=abc123",
            "burl": "https://example-dsp.com/billing?price=${AUCTION_PRICE}&id=abc123",
            "lurl": "https://example-dsp.com/loss?reason=${AUCTION_LOSS}&id=abc123",
            "adm": {
              "native": {
                "ver": "1.2",
                "assets": [
                  {
                    "id": 1,
                    "required": 1,
                    "video": {
                      "vasttag": "<VAST version=\"4.2\"><Ad id=\"12345\"><InLine><AdSystem>Example DSP</AdSystem><AdTitle>CTV Video Ad</AdTitle><Impression><![CDATA[https://example-dsp.com/impression?id=abc123]]></Impression><Creatives><Creative><Linear><Duration>00:00:15</Duration><MediaFiles><MediaFile delivery=\"progressive\" type=\"video/mp4\" width=\"1920\" height=\"1080\" bitrate=\"5000\"><![CDATA[https://cdn.example-dsp.com/videos/ctv-ad-1080p.mp4]]></MediaFile></MediaFiles><VideoClicks><ClickThrough><![CDATA[https://advertiser.com/landing]]></ClickThrough><ClickTracking><![CDATA[https://example-dsp.com/click?id=abc123]]></ClickTracking></VideoClicks></Linear></Creative></Creatives></InLine></Ad></VAST>"
                    }
                  }
                ],
                "link": {
                  "url": "https://advertiser.com/landing",
                  "clicktrackers": ["https://example-dsp.com/click?id=abc123"]
                },
                "eventtrackers": [
                  { "event": 1, "method": 1, "url": "https://example-dsp.com/impression?id=abc123" },
                  { "event": 1, "method": 2, "url": "https://example-dsp.com/js/tracker.js" },
                  { "event": 2, "method": 1, "url": "https://example-dsp.com/viewable?id=abc123" }
                ]
              }
            },
            "adomain": ["advertiser.com"],
            "bundle": "com.advertiser.app",
            "cid": "campaign-123",
            "crid": "creative-456",
            "cat": ["IAB1-1"],
            "api": 8,
            "protocol": 13,
            "w": 1920,
            "h": 1080,
            "exp": 9000
          }
        ],
        "seat": "example-dsp-seat-1"
        "bidid": "response-bid-id-abc123",
        "cur": "USD"
      }
}
```

Menu Ad - Video Only - OpenRTB Response - Native Payload - VAST

```xml
<VAST version="4.2">
  <Ad id="12345">
    <InLine>
      <AdSystem>Example DSP</AdSystem>
      <AdTitle>CTV Video Ad</AdTitle>
      <Impression><![CDATA[https://example-dsp.com/impression?id=abc123]]></Impression>
      <Creatives>
        <Creative>
          <Linear>
            <Duration>00:00:15</Duration>
            <MediaFiles>
              <MediaFile delivery="progressive" type="video/mp4" width="1920" height="1080" bitrate="5000">
                <![CDATA[https://cdn.example-dsp.com/videos/ctv-ad-1080p.mp4]]>
              </MediaFile>
            </MediaFiles>
            <VideoClicks>
              <ClickThrough><![CDATA[https://advertiser.com/landing]]></ClickThrough>
              <ClickTracking><![CDATA[https://example-dsp.com/click?id=abc123]]></ClickTracking>
            </VideoClicks>
          </Linear>
        </Creative>
      </Creatives>
    </InLine>
  </Ad>
</VAST>
```

Menu Ad - Image Only - OpenRTB Request

```json
{
    "id": "a3c1f842-6e7d-4b91-a5c8-9d2e0f7b6c34",
    "imp": [
      {
        "id": "1",
        "native": {
          "request": {
            "native": {
              "ver": "1.2",
              "context": 1,
              "contextsubtype": 10,
              "plcmttype": 1,
              "assets": [
                {
                  "id": 1,
                  "required": 1,
                  "img": {
                    "type": 3,
                    "w": 300,
                    "h": 250,
                    "mimes": ["image/jpeg", "image/png"]
                  }
                }
              ],
              "eventtrackers": [
                { "event": 1, "methods": [1, 2] },
                { "event": 2, "methods": [1] }
              ]
            }
          },
          "ver": "1.2"
        },
        "instl": 0,
        "tagid": "native-image-ad-unit-1",
        "bidfloor": 5.0,
        "bidfloorcur": "USD",
        "secure": 1,
        "exp": 9000
      }
    ],
    "app": {
      "id": "159384962",
      "name": "Example CTV App",
      "bundle": "com.example.ctvapp",
      "storeurl": "https://example.com/app",
      "cat": ["IAB1"],
      "publisher": {
        "id": "82524850",
        "name": "Example Publisher"
      }
    },
    "device": {
      "ua": "Mozilla/5.0 (Linux; SMART-TV 1.0) AppleWebKit/537.36 (KHTML, like Gecko) SMART-TV/4.0 TV Safari/537.36",
      "geo": {
        "country": "USA",
        "region": "CA",
        "city": "San Francisco"
      },
      "ip": "192.168.1.1",
      "devicetype": 3,
      "make": "Example TV",
      "model": "Smart TV",
      "os": "Example TV OS",
      "osv": "6.5",
      "w": 1920,
      "h": 1080,
      "ifa": "2d255aff-4389-418e-8f7b-3ff06d7e8901",
      "connectiontype": 1
    },
    "user": {
      "id": "user-12345"
    },
    "at": 2,
    "tmax": 250,
    "cur": ["USD"],
    "bcat": ["IAB25", "IAB26"],
    "badv": ["competitor.com"]
  }
```

Menu Ad - Image Only - OpenRTB Response

```json
{
    "id": "a3c1f842-6e7d-4b91-a5c8-9d2e0f7b6c34",
    "seatbid": [
      {
        "bid": [
          {
            "id": "bid-1",
            "impid": "1",
            "price": 7.50,
            "nurl": "https://example-dsp.com/win?price=${AUCTION_PRICE}&id=def456",
            "burl": "https://example-dsp.com/billing?price=${AUCTION_PRICE}&id=def456",
            "lurl": "https://example-dsp.com/loss?reason=${AUCTION_LOSS}&id=def456",
            "adm": {
              "native": {
                "ver": "1.2",
                "assets": [
                  {
                    "id": 1,
                    "required": 1,
                    "img": {
                      "url": "https://cdn.example-dsp.com/images/ctv-ad-300x250.jpg",
                      "w": 300,
                      "h": 250
                    }
                  }
                ],
                "link": {
                  "url": "https://advertiser.com/landing",
                  "clicktrackers": ["https://example-dsp.com/click?id=def456"]
                },
                "eventtrackers": [
                  { "event": 1, "method": 1, "url": "https://example-dsp.com/impression?id=def456" },
                  { "event": 1, "method": 2, "url": "https://example-dsp.com/js/tracker.js" },
                  { "event": 2, "method": 1, "url": "https://example-dsp.com/viewable?id=def456" }
                ]
              }
            },
            "adomain": ["advertiser.com"],
            "bundle": "com.advertiser.app",
            "cid": "campaign-123",
            "crid": "creative-789",
            "cat": ["IAB1-1"],
            "w": 300,
            "h": 250,
            "exp": 9000
          }
        ],
        "seat": "example-dsp-seat-1"
      }
    ],
    "bidid": "response-bid-id-67890",
    "cur": "USD"
}
```

Menu Ad - Mixed - OpenRTB Request

```json
{
    "id": "f7b2d914-8c3a-4e61-b0f5-6a9d1c8e2b47",
    "imp": [
      {
        "id": "1",
        "native": {
          "request": {
            "native": {
              "ver": "1.2",
              "context": 1,
              "contextsubtype": 10,
              "plcmttype": 3,
              "assets": [
                {
                  "id": 1,
                  "required": 1,
                  "img": {
                    "type": 3,
                    "w": 1920,
                    "h": 360,
                    "mimes": ["image/jpeg", "image/png"]
                  }
                },
                {
                  "id": 2,
                  "required": 0,
                  "video": {
                    "mimes": ["video/mp4"],
                    "minduration": 5,
                    "maxduration": 30,
                    "protocols": [3, 6, 7, 8, 11, 12, 13, 14, 15, 16],
                    "w": 1920,
                    "h": 1080,
                    "placement": 2,
                    "linearity": 1,
                    "skip": 0,
                    "playbackmethod": [2, 9],
                    "api": [8, 9]
                  }
                }
              ],
              "eventtrackers": [
                { "event": 1, "methods": [1, 2] },
                { "event": 2, "methods": [1] }
              ]
            }
          },
          "ver": "1.2"
        },
        "instl": 0,
        "tagid": "native-mixed-ad-unit-1",
        "bidfloor": 5.0,
        "bidfloorcur": "USD",
        "secure": 1,
        "exp": 9000
      }
    ],
    "app": {
      "id": "159384962",
      "name": "Example CTV App",
      "bundle": "com.example.ctvapp",
      "storeurl": "https://example.com/app",
      "cat": ["IAB1"],
      "publisher": {
        "id": "82524850",
        "name": "Example Publisher"
      }
    },
    "device": {
      "ua": "Mozilla/5.0 (Linux; SMART-TV 1.0) AppleWebKit/537.36 (KHTML, like Gecko) SMART-TV/4.0 TV Safari/537.36",
      "geo": {
        "country": "USA",
        "region": "NY",
        "city": "New York"
      },
      "ip": "192.168.1.1",
      "devicetype": 3,
      "make": "Example TV",
      "model": "Smart TV",
      "os": "Example TV OS",
      "osv": "6.5",
      "w": 1920,
      "h": 1080,
      "ifa": "2d255aff-4389-418e-8f7b-3ff06d7e8901",
      "connectiontype": 1
    },
    "user": {
      "id": "user-12345"
    },
    "at": 2,
    "tmax": 250,
    "cur": ["USD"],
    "bcat": ["IAB25", "IAB26"],
    "badv": ["competitor.com"]
  }
```

Menu Ad - Mixed - OpenRTB Response
```
{
  "id": "a3c1f842-6e7d-4b91-a5c8-9d2e0f7b6c34",
  "seatbid": [
    {
      "bid": [
        {
          "id": "bid-1",
          "impid": "1",
          "price": 7.50,
          "nurl": "https://example-dsp.com/win?price=${AUCTION_PRICE}&id=def456",
          "burl": "https://example-dsp.com/billing?price=${AUCTION_PRICE}&id=def456",
          "lurl": "https://example-dsp.com/loss?reason=${AUCTION_LOSS}&id=def456",
          "adm": {
            "native": {
              "ver": "1.2",
              "assets": [
                {
                  "id": 1,
                  "required": 1,
                  "img": {
                    "url": "https://cdn.example-dsp.com/images/ctv-banner-1920x360.jpg",
                    "w": 1920,
                    "h": 360
                  }
                },
                {
                  "id": 2,
                  "required": 0,
                  "video": {
                    "vasttag": "<VAST version=\"4.2\"><Ad id=\"12345\"><InLine><AdSystem>Example DSP</AdSystem><AdTitle>CTV Video Ad</AdTitle><Impression><![CDATA[https://example-dsp.com/impression?id=ghi789]]></Impression><Creatives><Creative><Linear><Duration>00:00:15</Duration><MediaFiles><MediaFile delivery=\"progressive\" type=\"video/mp4\" width=\"1920\" height=\"1080\" bitrate=\"5000\"><![CDATA[https://cdn.example-dsp.com/videos/ctv-ad-1080p.mp4]]></MediaFile></MediaFiles><VideoClicks><ClickThrough><![CDATA[https://advertiser.com/landing]]></ClickThrough><ClickTracking><![CDATA[https://example-dsp.com/click?id=ghi789]]></ClickTracking></VideoClicks></Linear></Creative></Creatives></InLine></Ad></VAST>"
                  }
                }
              ],
              "link": {
                "url": "https://advertiser.com/landing",
                "clicktrackers": ["https://example-dsp.com/click?id=ghi789"]
              },
              "eventtrackers": [
                { "event": 1, "method": 1, "url": "https://example-dsp.com/impression?id=ghi789" },
                { "event": 1, "method": 2, "url": "https://example-dsp.com/js/tracker.js" },
                { "event": 2, "method": 1, "url": "https://example-dsp.com/viewable?id=ghi789" }
              ]
            }
          },
          "adomain": ["advertiser.com"],
          "bundle": "com.advertiser.app",
          "cid": "campaign-123",
          "crid": "creative-789",
          "cat": ["IAB1-1"],
          "w": 300,
          "h": 250,
          "exp": 9000
        }
      ],
      "seat": "example-dsp-seat-1"
    }
  ],
  "bidid": "response-bid-id-67890",
  "cur": "USD"
}
```

Menu Ad - Mixed - OpenRTB Response - Native Payload - VAST

```xml
<VAST version="4.2">
  <Ad id="12345">
    <InLine>
      <AdSystem>Example DSP</AdSystem>
      <AdTitle>CTV Video Ad</AdTitle>
      <Impression><![CDATA[https://example-dsp.com/impression?id=ghi789]]></Impression>
      <Creatives>
        <Creative>
          <Linear>
            <Duration>00:00:15</Duration>
            <MediaFiles>
              <MediaFile delivery="progressive" type="video/mp4" width="1920" height="1080" bitrate="5000">
                <![CDATA[https://cdn.example-dsp.com/videos/ctv-ad-1080p.mp4]]>
              </MediaFile>
            </MediaFiles>
            <VideoClicks>
              <ClickThrough><![CDATA[https://advertiser.com/landing]]></ClickThrough>
              <ClickTracking><![CDATA[https://example-dsp.com/click?id=ghi789]]></ClickTracking>
            </VideoClicks>
          </Linear>
        </Creative>
      </Creatives>
    </InLine>
```

Pause & Screensaver Ad(s)

The following examples demonstrate two complete request-response-VAST InLine flows for (1) full-screen static image and (2) cinemagraph. Screensaver scenarios follow the same structure with the appropriate placement and playback method values. 

Example 1: Screensaver - Static Image Pause Ad (Fullscreen)

A FAST (Free Ad-Supported Streaming TV) publisher XYZ sends a Screensaver Ad opportunity. The device is idle on the channel guide screen. The DSP responds with a JPEG image at 1920x1080 creative. No Duration element, no quartile tracking. Measurement relies on creativeView and overlayViewDuration.

Bid Request
```json
{
  "id": "fast-screen-004",
  "at": 1,
  "imp": [
	{
  	"id": "imp-1",
  	"video": {
    	"skip": 0,
    	"w": 1920,
    	"h": 1080,
    	"linearity": 2,
    	"plcmt": 6,
    	"pos":7,
    	"playbackmethod": [11],
    	"battr": [22,23],
    	"mimes": ["image/jpeg", "image/png"],

    	"protocols": [7, 8]
  	},
  	"bidfloor": 5.00,
  	"bidfloorcur": "USD",
  	"secure": 1,
	}
  ],
  "app": {
	"bundle": "tv.XYZ.android",
	"name": "XYZ TV",
	"publisher": {
  	"id": "pub-xyz-001",
  	"name": "XYZ Streaming"
	}
  },
  "device": {
	"ua": "Mozilla/5.0 (Linux; Android 12; BRAVIA XR)",
	"devicetype": 3,
	"ifa": "d4e5f6a7-b8c9-0123-defa-234567890123",
	"lmt": 0,
	"make": "Sony",
	"model": "XR-55A95K",
	"w": 3840,
	"h": 2160
  },
  "user": {},
  "regs": {}
}
```
 
Bid Response
```json
{
  "id": "fast-screen-004",
  "seatbid": [
	{
  	"seat": "seat-dsp-delta",
  	"bid": [
    	{
      	"id": "bid-static-004",
      	"impid": "imp-1",
      	"price": 7.50,
      	"adm": "<VAST version=\"4.2\"><!-- Static Image NonLinear VAST --></VAST>",
		"attr": [19,21]
      	"adid": "ad-brand-w-static",
      	"adomain": ["brand-w.com"],
      	"crid": "cr-static-004",
      	"w": 1920,
      	"h": 1080,
      	"mtype": 2,
      	"burl": "https://dsp-delta.example.com/billing?imp=${AUCTION_IMP_ID}&price=${AUCTION_PRICE}",
      	"lurl": "https://dsp-delta.example.com/loss?imp=${AUCTION_IMP_ID}&reason=${AUCTION_LOSS}",

    	}
  	]
	}
  ],
  "cur": "USD"
}
```

```xml
VAST 
<VAST version="4.2">
  <Ad id="pause-static-001">
    <InLine>
      <AdSystem version="1.0">ExampleAdServer</AdSystem>
      <AdTitle>Brand X ScreensaverAd - Static</AdTitle>
      <Impression><![CDATA[https://track.example.com/imp?id=001]]></Impression>
      <AdServingId>ServerName-a1b2c3d4-e5f6-7890</AdServingId>
      <Creatives>
        <Creative id="creative-static-001" adId="AD-ID-001">
          <UniversalAdId idRegistry="ad-id.org">CNPA0484000H</UniversalAdId>

          <NonLinearAds>
            <NonLinear width="1920" height="1080"
                scalable="true" maintainAspectRatio="true">
              <StaticResource creativeType="image/jpeg">
                <![CDATA[https://cdn.example.com/pause/brand_x_1920x1080.jpg]]>
              </StaticResource>
              <NonLinearClickThrough>
                <![CDATA[https://brand-x.example.com/landing]]>
              </NonLinearClickThrough>
              <NonLinearClickTracking>
                <![CDATA[https://track.example.com/click?id=001]]>
              </NonLinearClickTracking>
            </NonLinear>
            <TrackingEvents>
              <Tracking event="creativeView">
                <![CDATA[https://track.example.com/creativeView?id=001]]>
              </Tracking>
              <Tracking event="overlayViewDuration">
                <![CDATA[https://track.example.com/overlayView?id=001]]>
              </Tracking>
              <Tracking event="close">
                <![CDATA[https://track.example.com/close?id=001]]>
              </Tracking>
            </TrackingEvents>
          </NonLinearAds>
        </Creative>
      </Creatives>
      <Extensions>
		<Extension type="pos" ext="adcom">
		  <pos>7</pos>
		</Extension>

		<Extension type="attr" ext="adcom">
		  <attr>21</attr>
		</Extension>
      </Extensions>
    </InLine>
  </Ad>
</VAST>
```

Example 2: Cinemagraph Pause Ad (Fullscreen, Cinemagraph)
A broadcaster ZYX sends a Pause Ad opportunity triggered by the viewer pausing a live sports stream. The ad placement is fullscreen (1920x1080). The DSP responds with a motion-image MP4 with subtle animation and no audio. Loops continuously until the user resumes playback. Duration enables full quartile tracking.

Bid Request
```json
{
  "id": "br-pause-001",
  "at": 1,
  "imp": [
	{
  	"id": "imp-1",
  	"video": {
    	"linearity": 2,
    	"plcmt": 5,
    	"pos":7,
    	"playbackmethod": [9],
    	"mimes": ["video/mp4"],
    	"battr": [23],
       "skip": 0,
    	"w": 1920,
    	"h": 1080,
    	"minduration": 30,
    	"maxduration": 30,
    	"protocols": [7, 8]
  	},
  	"bidfloor": 15.00,
  	"bidfloorcur": "USD",
  	"secure": 1,
  	"exp": 300,
	}
  ],
  "app": {
	"bundle": "com.zyx.streamingapp",
	"name": "ZYX",
	"publisher": {
  	"id": "pub-zyxu-001",
  	"name": "ZYXUniversal"
	},
	"content": {
  	"id": "content-12345",
  	"title": "NFL Sunday Night Football",
  	"livestream": 1,
  	"cat": ["IAB17-12"],
  	"genres": [605],
  	"gtax": 1
	}
  },
  "device": {
	"ua": "Mozilla/5.0 (SMART-TV; LG; webOS/6.0)",
	"devicetype": 3,
	"ifa": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
	"lmt": 0,
	"w": 1920,
	"h": 1080
  },
  "user": {},
  "regs": {
	"ext": { "us_privacy": "1YNN" }
  }
}
```
 
Bid Response
```json
{
  "id": "br-pause-001",
  "seatbid": [
	{
  	"seat": "seat-dsp-alpha",
  	"bid": [
    	{
      	"id": "bid-cine-001",
      	"impid": "imp-1",
      	"price": 22.50,
      	"adm": "<VAST version=\"4.2\"><!-- Cinemagraph NonLinear VAST --></VAST>",
      	"adid": "ad-brand-x-cine",
      	"adomain": ["brand-x.com"],
      	"crid": "cr-cinemagraph-001",
      	"w": 1920,
      	"h": 1080,
      	"mtype": 2,
      	"attr": [19,22],
      	"dur": 10,
      	"burl": "https://dsp.example.com/billing?imp=${AUCTION_IMP_ID}&price=${AUCTION_PRICE}",
      	"lurl": "https://dsp.example.com/loss?imp=${AUCTION_IMP_ID}&reason=${AUCTION_LOSS}",

    	}
  	]
	}
  ],
  "cur": "USD"
}
```

VAST
```xml
<VAST version="4.2">
  <Ad id="pause-cine-003">
    <InLine>
      <AdSystem version="1.0">ExampleAdServer</AdSystem>
      <AdTitle>Brand Z Pause Ad - Cinemagraph</AdTitle>
      <Impression><![CDATA[https://track.example.com/imp?id=003]]></Impression>
      <AdServingId>ServerName-c3d4e5f6-a7b8-9012</AdServingId>
      <Creatives>
        <Creative id="creative-cine-003" adId="AD-ID-003">
          <UniversalAdId idRegistry="ad-id.org">CNPA0484002H</UniversalAdId>

          <NonLinearAds>
            <NonLinear width="1920" height="1080"
                scalable="true" maintainAspectRatio="true"
>

              <MediaFiles>
                <MediaFile delivery="progressive" type="video/mp4"
                    width="1920" height="1080" codec="H.264"
                    bitrate="8000">
                  <![CDATA[https://cdn.example.com/pause/brand_z_cine_1080p.mp4]]>
                </MediaFile>
                <MediaFile delivery="progressive" type="video/mp4"
                    width="1280" height="720" codec="H.264"
                    bitrate="4000">
                  <![CDATA[https://cdn.example.com/pause/brand_z_cine_720p.mp4]]>
                </MediaFile>
              </MediaFiles>
              <Duration>00:00:30</Duration>
              <NonLinearClickThrough>
                <![CDATA[https://brand-z.example.com/landing]]>
              </NonLinearClickThrough>
            </NonLinear>
            <TrackingEvents>
              <Tracking event="creativeView">
                <![CDATA[https://track.example.com/creativeView?id=003]]>
              </Tracking>
              <Tracking event="start">
                <![CDATA[https://track.example.com/start?id=003]]>
              </Tracking>
              <Tracking event="firstQuartile">
                <![CDATA[https://track.example.com/firstQ?id=003]]>
              </Tracking>
              <Tracking event="midpoint">
                <![CDATA[https://track.example.com/midpoint?id=003]]>
              </Tracking>
              <Tracking event="thirdQuartile">
                <![CDATA[https://track.example.com/thirdQ?id=003]]>
              </Tracking>
              <Tracking event="complete">
                <![CDATA[https://track.example.com/complete?id=003]]>
              </Tracking>
              <Tracking event="close">
                <![CDATA[https://track.example.com/close?id=003]]>
              </Tracking>
            </TrackingEvents>
          </NonLinearAds>
        </Creative>
      </Creatives>
      <Extensions>

		<Extension type="pos" ext="adcom">
		  <pos>7</pos>
		</Extension>

		<Extension type="attr" ext="adcom">
		  <attr>22</attr>
		</Extension>
        </Extension>

  </Ad>
</VAST>
```

In-Scene Ads 

Bid Request
```json
{
	"id":"req-iscene-001",
	"imp":[
		{
			"id":"1",
			"bidfloor":9.5,
			"video":{
				"plcmt":9,
				"linearity":2,
			    	"w": 1400,
			    	"h": 400,
				"mimes":[
					"image/png",
					"image/jpeg",
					"video/mp4"
				],
				"playbackmethod":[
					8
				]
			}
				
					]
				}
		},
	],
	"app":{
		"id":"example.ctv.app",
		"name":"Publisher CTV App",
		"publisher":{
			"id":"pub-123",
			"name":"Publisher"
		},
		"content":{
			"id":"ep-456",
			"title":"Drama Episode",
			"series":"Drama Series",
			"season":"2",
			"episode":5,
			"cat":[
				"IAB1-6"
			]
		}
	}
}
```

Bid Response

```json
{
  "id": "req-iscene-001",
  "seatbid": [
    {
      "bid": [
        {
          "impid": "1",
          "price": 12.1,
          "adm": "<VAST>...</VAST>",
	    "mtype": 2
        }
      ]
    }
  ]
}
```

VAST in Bid Response
Static Image

```xml
<?xml version="1.0" encoding="UTF-8"?>
<VAST xmlns="http://www.iab.com/VAST" version="4.2">
<Ad id="ISCENE-23918">
<InLine>
<AdSystem>ADSERVER</AdSystem>
<AdTitle><![CDATA[ISCENE-23918]]></AdTitle>
<Impression id="Imp"><![CDATA[IMPRESSION_COUNT]]></Impression>
<Error><![CDATA[ERROR_TRACKER]]></Error>
<Creatives>
<Creative id="1">
<UniversalAdId idRegistry="your.registry">982765</UniversalAdId>
<NonLinearAds>
<TrackingEvents>
<Tracking event="start"><![CDATA[START]]></Tracking>
<Tracking event="firstQuartile"><![CDATA[FIRST]]></Tracking>
<Tracking event="midpoint"><![CDATA[MID]]></Tracking>
<Tracking event="thirdQuartile"><![CDATA[THIRD]]></Tracking>
<Tracking event="complete"><![CDATA[COMPLETE]]></Tracking>
</TrackingEvents>

<NonLinear width="1400" height="400" scalable="true" maintainAspectRatio="true">
<Duration>00:00:05</Duration>
<StaticResource id="iscene" creativeType="image/png">
<![CDATA[https://your.cdn.net/creative/billboard/ISCENE-23918.png]]>
</StaticResource>
</NonLinear>
</NonLinearAds>
</Creative>
</Creatives>
</InLine>
</Ad>
</VAST>
```

Video Response

```xml
<?xml version="1.0" encoding="UTF-8"?>
<VAST xmlns="http://www.iab.com/VAST" version="4.2">
<Ad id="ISCENE-23918">
<InLine>
<AdSystem>ADSERVER</AdSystem>
<AdTitle><![CDATA[ISCENE-23918]]></AdTitle>
<Impression id="Imp"><![CDATA[IMPRESSION_COUNT]]></Impression>
<Error><![CDATA[ERROR_TRACKER]]></Error>
<Creatives>
<Creative id="1">
<UniversalAdId idRegistry="your.registry">982765</UniversalAdId>
<NonLinearAds>
<TrackingEvents>
<Tracking event="start"><![CDATA[START]]></Tracking>
<Tracking event="firstQuartile"><![CDATA[FIRST]]></Tracking>
<Tracking event="midpoint"><![CDATA[MID]]></Tracking>
<Tracking event="thirdQuartile"><![CDATA[THIRD]]></Tracking>
<Tracking event="complete"><![CDATA[COMPLETE]]></Tracking>
</TrackingEvents>
<NonLinear width="1400" height="400" scalable="true" maintainAspectRatio="true">
<Duration>00:00:05</Duration>
<MediaFiles>
<MediaFile delivery="progressive" type="video/mp4" width="1400" height="400" bitrate="6000">
<![CDATA[https://cdn.example.com/creatives/iscene_video_asset.mp4]]>
</MediaFile>
</MediaFiles>
</NonLinear>
</NonLinearAds>
</Creative>
</Creatives>
</InLine>
</Ad>
</VAST>
```


Squeezeback
Example: Squeezeback Ad (L-Banner with content in Upper Right) 

Bid Request
```json
{
  "id": "squeezeback001",
  "at": 1,
  "imp": [
	{
  	"id": "imp-1",
  	"video": {
    	"linearity": 2,
    	"plcmt": 8,
    	"pos":16,
    	"playbackmethod": [2],
    	"mimes": ["img/png"],
    	
       "skip": 0,
    	"w": 1920,
    	"h": 1080,
    	"minduration": 15,
    	"maxduration": 15,
    	"protocols": [7, 8]
  	},
  	"bidfloor": 15.00,
  	"bidfloorcur": "USD",
  	"secure": 1,
  	"exp": 300,
	}
  ],
  "app": {
	"bundle": "com.zyx.streamingapp",
	"name": "ZYX",
	"publisher": {
  	"id": "pub-zyxu-001",
  	"name": "ZYXUniversal"
	},
	"content": {
  	"id": "content-12345",
  	"title": "NFL Sunday Night Football",
  	"livestream": 1,
  	"cat": ["IAB17-12"],
  	"genres": [605],
  	"gtax": 1
	}
  },
  "device": {
	"ua": "Mozilla/5.0 (SMART-TV; LG; webOS/6.0)",
	"devicetype": 3,
	"ifa": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
	"lmt": 0,
	"w": 1920,
	"h": 1080
  },
  "user": {},
  "regs": {
	"ext": { "us_privacy": "1YNN" }
  }
}
```
 
Bid Response

```json
{
  "id": "squeezeback001",
  "seatbid": [
	{
  	"seat": "seat-dsp-alpha",
  	"bid": [
    	{
      	"id": "bid-cine-001",
      	"impid": "imp-1",
      	"price": 22.50,
      	"adm": "<VAST version=\"4.2\"><!-- NonLinear VAST --></VAST>",
      	"adid": "ad-brand-x-cine",
      	"adomain": ["brand-x.com"],
     
      	"w": 1920,
      	"h": 1080,
      	"mtype": 2,
      	"dur": 10,
      	"burl": "https://dsp.example.com/billing?imp=${AUCTION_IMP_ID}&price=${AUCTION_PRICE}",
      	"lurl": "https://dsp.example.com/loss?imp=${AUCTION_IMP_ID}&reason=${AUCTION_LOSS}",

    	}
  	]
	}
  ],
  "cur": "USD"
}
```

VAST Response

```xml
<?xml version="1.0" encoding="UTF-8"?>
<VAST xmlns="http://www.iab.com/VAST" version="4.2">
    <Ad id="123456">
        <InLine>
            <AdSystem>ADSERVER</AdSystem>
            <AdTitle><![CDATA[123456]]></AdTitle>
            <Impression id="Imp"><![CDATA[IMPRESSION_COUNT]]></Impression>
            <Error><![CDATA[ERROR_TRACKER]]></Error>
            <Creatives>
                <Creative id="1">
			          <UniversalAdId idRegistry="your.registry">98765</UniversalAdId>
                <NonLinearAds>
			            <TrackingEvents>
                            <Tracking event="start"><![CDATA[START]]></Tracking>
                            <Tracking event="firstQuartile"><![CDATA[FIRST]]></Tracking>
                            <Tracking event="midpoint"><![CDATA[MID]]></Tracking>
                            <Tracking event="thirdQuartile"><![CDATA[THIRD]]></Tracking>
                            <Tracking event="complete"><![CDATA[COMPLETE]]></Tracking>
                        </TrackingEvents>
                        <NonLinear width="1280" height="720" scalable="true" maintainAspectRatio="true">
     			            	<Duration>00:00:10</Duration>
<StaticResource id="atv" creativeType="image/png">
<![CDATA[https://your.cdn.net/creative/Lbanner/123456.png]]>
</StaticResource>
<NonLinearClickThrough>
<![CDATA[https://your.landingpage.net/click?ad=123456]]>
</NonLinearClickThrough>
                </NonLinear>
                 </NonLinearAds>
                </Creative>
            </Creatives>
        <Extensions>
		<Extension type="plcmt" ext="adcom">
		  <plcmt>8</plcmt>
		</Extension>
		<Extension type="pos" ext="adcom">
		  <pos>16</pos>
		</Extension>
		<Extension type="playbackmethod" ext="adcom">
		  <playbackmethod>2</playbackmethod>
		</Extension>
        </Extension>
        </InLine>
    </Ad>
</VAST>
```

Overlay 
Example: Overlay Ad
Bid Request
```json
{  "id": "overlay001",
  "at": 1,
 "imp": [ 
{
  	"id": "imp-1",
  	"video": {
    	"linearity": 2,
    	"plcmt": 7,
    	"pos":5,
    	"playbackmethod": [2],
    	"mimes": ["video/mp4, image/png"],
      "skip": 0,
    	"w": 1920,
    	"h": 1080,
    	"minduration": 10,
    	"maxduration": 10,
    	"protocols": [7, 8]
  	},
  	"bidfloor": 15.00,
  	"bidfloorcur": "USD",
  	"secure": 1,
  	"exp": 300,
	}
  ],
  "app": {
	"bundle": "com.zyx.streamingapp",
	"name": "ZYX",
	"publisher": {
  	"id": "pub-zyxu-001",
  	"name": "ZYXUniversal"
	},
	"content": {
  	"id": "content-12345",
  	"title": "NFL Sunday Night Football",
  	"livestream": 1,
  	"cat": ["IAB17-12"],
  	"genres": [605],
  	"gtax": 1
	}
  },
  "device": {
	"ua": "Mozilla/5.0 (SMART-TV; LG; webOS/6.0)",
	"devicetype": 3,
	"ifa": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
	"lmt": 0,
	"w": 1920,
	"h": 1080
  },
  "user": {},
  "regs": {
	"ext": { "us_privacy": "1YNN" }
  }
}
```

Bid Response
```json
{
  "id": "overlay001",
  "seatbid": [
	{
  	"seat": "seat-dsp-alpha",
  	"bid": [
    	{
      	"id": "bid-cine-001",
      	"impid": "imp-1",
      	"price": 22.50,
      	"adm": "<VAST version=\"4.2\"><!-- NonLinear VAST --></VAST>",
      	"adid": "ad-brand-x-cine",
      	"adomain": ["brand-x.com"],
     
      	"w": 1920,
      	"h": 1080,
      	"mtype": 2,
      	"dur": 10,
      	"burl": "https://dsp.example.com/billing?imp=${AUCTION_IMP_ID}&price=${AUCTION_PRICE}",
      	"lurl": "https://dsp.example.com/loss?imp=${AUCTION_IMP_ID}&reason=${AUCTION_LOSS}",

    	}
  	]
	}
  ],
  "cur": "USD"
}
```


VAST - Video Response
```xml
<VAST version="INSERT VAST VERSION">
<Ad>
<InLine>
<AdSystem>INSERT AD SYSTEM HERE</AdSystem>
<AdTitle>INSERT CREATIVE NAME HERE</AdTitle>
<Impression/>
<Creatives>
<Creative>
<NonLinear>
<Duration>00:00:10</Duration>
<TrackingEvents/>
<MediaFiles>
<MediaFile height="INSERT AD HEIGHT HERE" width="INSERT AD WIDTH HERE" type="video/quicktime">
<![CDATA[INSERT .MOV FILE URL HERE]]>
</MediaFile>
</MediaFiles>
</NonLinear>
</Creative>
</Creatives>
</InLine>
</Ad>
</VAST>
```

```xml
<VAST version="4.2">
 <Ad id="ad_overlay_001">
   <InLine>
     <AdSystem version="1.0">ExampleAdServer</AdSystem>
     <AdTitle><![CDATA[Overlay Video Ad]]></AdTitle>
     <AdServingId>ServerName-a1b2c3d4-e5f6-7890</AdServingId>
     <Impression id="imp_main">
       <![CDATA[https://track.example.com/imp?id=overlay001]]>
     </Impression>
     <Creatives>
       <Creative id="cr_overlay_001" adId="AD-ID-OVL-001">
         <UniversalAdId idRegistry="ad-id.org">CNPA0484020H</UniversalAdId>
         <NonLinearAds>
            <NonLinear id="nl_overlay_001"
                      width="1920" height="324"
                      scalable="true"
                      maintainAspectRatio="true">
             <Duration>00:00:10</Duration>
             <MediaFiles>
               <MediaFile delivery="progressive"
                          type="video/webm"
                          width="1920" height="324"
                          codec="VP9"
                          bitrate="4000">
           <![CDATA[https://cdn.example.com/ads/overlay/brand_overlay_1920x1080.webm]]>
               </MediaFile>
             </MediaFiles>
             <NonLinearClickThrough>
               <![CDATA[https://advertiser.example.com/landing?ad=overlay001]]>
             </NonLinearClickThrough>
             <NonLinearClickTracking id="click_nl_001">
               <![CDATA[https://track.example.com/click?id=overlay001]]>
             </NonLinearClickTracking>
           </NonLinear>
           <TrackingEvents>
             <Tracking event="creativeView">
               <![CDATA[https://track.example.com/creativeView?id=overlay001]]>
             </Tracking>
             <Tracking event="start">
               <![CDATA[https://track.example.com/start?id=overlay001]]>
             </Tracking>
             <Tracking event="firstQuartile">
               <![CDATA[https://track.example.com/firstQ?id=overlay001]]>
             </Tracking>
             <Tracking event="midpoint">
               <![CDATA[https://track.example.com/midpoint?id=overlay001]]>
             </Tracking>
             <Tracking event="thirdQuartile">
               <![CDATA[https://track.example.com/thirdQ?id=overlay001]]>
             </Tracking>
             <Tracking event="complete">
               <![CDATA[https://track.example.com/complete?id=overlay001]]>
             </Tracking>
             <Tracking event="close">
               <![CDATA[https://track.example.com/close?id=overlay001]]>
             </Tracking>
             <Tracking event="overlayViewDuration">
<![CDATA[https://track.example.com/overlayViewDuration?id=overlay001]]>
             </Tracking>
           </TrackingEvents>
         </NonLinearAds>
       </Creative>
     </Creatives>
     <Extensions>
       <Extension type="plcmt" ext="adcom">
         <plcmt>7</plcmt>
       </Extension>
       <Extension type="pos" ext="adcom">
         <pos>14</pos>
       </Extension>
       <Extension type="playbackmethod" ext="adcom">
         <playbackmethod>2</playbackmethod>
       </Extension>
       <Extension type="attr" ext="adcom">
         <attr>20</attr>
       </Extension>
     </Extensions>
   </InLine>
 </Ad>
</VAST>
```

VAST Static Image Response

```xml
<?xml version="1.0" encoding="UTF-8"?>
<VAST xmlns="http://www.iab.com/VAST" version="4.2">
    <Ad id="123456">
        <InLine>
            <AdSystem>ADSERVER</AdSystem>
            <AdTitle><![CDATA[123456]]></AdTitle>
            <Impression id="Imp"><![CDATA[IMPRESSION_COUNT]]></Impression>
     <Error><![CDATA[ERROR_TRACKER]]></Error>
            <Creatives>
                <Creative id="1">
			<UniversalAdId idRegistry="your.registry">98765</UniversalAdId>
                    <NonLinearAds>
			     <TrackingEvents>
                            <Tracking event="start"><![CDATA[START]]></Tracking>
                            <Tracking event="firstQuartile"><![CDATA[FIRST]]></Tracking>
                            <Tracking event="midpoint"><![CDATA[MID]]></Tracking>
                            <Tracking event="thirdQuartile"><![CDATA[THIRD]]></Tracking>
                            <Tracking event="complete"><![CDATA[COMPLETE]]></Tracking>
                        </TrackingEvents>
                        <NonLinear width="300" 
height="250" 
scalable="true" 
maintainAspectRatio="true">	
			    	<Duration>00:00:10</Duration>
                        	<StaticResource id="atv" creativeType="image/png">      <![CDATA[https://your.cdn.net/creative/overlay/123456.png]]>
</StaticResource>
<NonLinearClickThrough>
<![CDATA[https://your.landingpage.net/click?ad=123456]]>
</NonLinearClickThrough>
                        </NonLinear>
                    </NonLinearAds>
                </Creative>
            </Creatives>
        </InLine>
    </Ad>
</VAST>
```
