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


	LinearAds:
	use <InteractiveCreativeFile apiFramework=”SIMID”> to declare the creative markup in <MediaFile> is SIMID 


	NonLinearAds:
	use <IFrameResource apiFramework=”SIMID”> to both declare SIMID and provide the creative markup.


## Purpose of VAST ext & Failure Cases
Advertisers should respect the values provided in the bid request by the publisher and ensure that any creative sent is a match. 
Publishers integrations should be resilient when receiving attributes or assets that they didn’t expect. They should ensure they have a way to adhere to their preferred user experience regardless of the creative received. 
