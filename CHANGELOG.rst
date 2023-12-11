v1.12.12
=========

New Features:
--------------

* Add access time to API and manifest proxy access logs (#250)
* Allow HEAD requests to manifest proxy (#246)

Bugfixes:
----------

* Fix gaps in DASH multi-period output (#248)
* Clean up manifest proxy debug logging (#241)

v1.12.11
=========

New Features:
--------------

* Enable full Manifest Edit functionality (#245)

v1.12.10
=========

New Features:
--------------

* Improve handling of errors in HLS media sequence counting when a transition to a live source does not work as expected (#243)
* Change transition worker to continuously check manifests to ensure correct state is maintained (#238)

Bugfixes:
----------

* Fix DASH incorrect period ID and start time when producing multi-period output on base channel (#240)
* Correctly set MPD@minimumUpdatePeriod and MPD@timeShiftBufferDepth in dynamic manifests (#236)

v1.12.9
========

Bugfixes:
----------

* Fix creation of check transition task for live transitions (#234)

v1.12.8
========

Breaking Changes:
------------------

* Output separate DASH periods for ad breaks (#203)

  * Database changes related to this feature may break DASH output for existing channels and they will need to be recreated.

New Features:
--------------

* Add MANIFEST_PROXY_HOST option for transition worker to connect to remote manifest proxy (#225)

Bugfixes:
----------

* Properly handle request with vend in the future by capping it to current wallclock time (#229)
* Set correct Expires and Cache-Control: max-age headers when serving a live manifest, based on the expected next segment time (#226)

v1.12.5
========

Bugfixes:
----------

* Fix HLS keyframe playlist segment URLs (#224)
* Update HLS media sequence to account for extra segments added for ad insertion or playlist looping (#223)

v1.12.4
========

New Features:
--------------

* Allow custom RabbitMQ connection string for Celery by setting CELERY_BROKER environment variable (#221)
* Support optional hls_minimum_fragment_length configuration in live source SMIL head (#217)

Bugfixes:
----------

* Correctly set start and end times on EXT-X-DATERANGE tags not attached to an EXT-INF (#222)
* Set correct timeShiftBufferDepth on dynamic DASH manifests with fixed vbegin (#219)
* Give more informative error message when requesting manifest with invalid vbegin, vend, or t query parameter (#218)

v1.12.3
========

Bugfixes:
----------

* Fix #EXT-X-MEDIA-SEQUENCE for streams with variable EXTINF durations (#216)

v1.12.2
========

New Features:
--------------

* Allow configuration of external Redis and RabbitMQ (#211)

Bugfixes:
----------

* Fix incorrect headers set on manifest proxy response (#206)

v1.12.1
========

Bugfixes:
----------

* Fixed issue preventing transition creation when maximum number of channel is reached (#201)

v1.12.0
========

Bugfixes:
----------

* Fix DASH BaseURL calculation for transitions (#199)

v1.11.24
=========

Breaking Changes:
------------------

* Error out on transition if base channel does not exists (#185)
* Reuse ismls as much as possible to improve cache efficiency of multiple channels with shared playlists (#147)

New Features:
--------------

* Error out on transition creation if transition time is < vod2live start time (#186)
* Transitions are aborted if not ready before the scheduled transition time (#62)
* Automatic deletion of old playlists (configurable, defaults to 7 days retention) (#50)

Bugfixes:
----------

* LOG_LEVEL values "warn" and "alert" now work correctly (#190)
* Improve SMIL parsing to properly handle optional attributes on EventStream and Event elements (#184)

v1.11.23
=========

Breaking Changes:
------------------

* API Manifest Edit endpoints path renamed from .../pipelines/formats/{format} to .../pipelines/{format} (#174)
* API log endpoints path renamed from .../{channel}/log to .../{channel}/logs (#173)
* API transition endpoints path renamed from .../{channel}/{transition_time} to .../{channel}/transitions/{transition_time} (#172)
* API endpoints paths (all) renamed from /channel to /channels (#171)

New Features:
--------------

* Switch uvicorn event loop to use uvloop (#176)
* Switch from Alpine Linux to Ubuntu 22.04 (#175)
* Remix timeout is now handled correctly and configurable (#170)
* Added working Manifest Edit example in Swagger UI (#164)
* Adaptation Set IDs in DASH manifests are now unsigned integers instead of strings (#159)
* A single env file now collects all Virtual Channel environmental variables used for configuration (#150)

Bugfixes:
----------

* DASH manifest publishTime is now wallclock-based instead of starting from zero (#168)
* Fix HLS key change signalling on transition (#162)

v1.11.22
=========

New Features:
--------------

* Add option to delay VOD2Live outputs to align media timeline with Live sources (#163)
* Manifest Proxy now integrates Manifest Edit functionalities (#154)

v1.11.21
=========

New Features:
--------------

* Add support for DRM paramGroups and HLS variantSets to SMIL parser (#149)
* Reduced README content, now uses rst format and links to Unified doc pages (#145)
* Code obfuscation (#132)
* A license with Virtual Channel specific flags is now required (#126)

Bugfixes:
----------

* Transitions are now refused if channel creation was not successful (#152)
* Fixed version number tag in /version endpoint and API doc page (#144)

v1.11.20
=========

Breaking Changes:
------------------

* #84: The response of GET /channel/{channel}/transitions endpoint has changed in a non-backwards compatible way. It now returns a dictionary including details on status and related smil. Filtering on status is supported.

New Features:
--------------

* #124: Test if playback works when transitioning across playlist with different encryption/drm settings
* #120: The GET /channel/{channel}/transitions endpoint now support time-based queries using the "begin" and "end" query parameters.
* #119: Improve delete API and file tracking
* #83: The GET /channel endpoint now only reports channels created with PUT /channel/{channel_name} requests.
* #73: If a job is submitted that can reuse existing remix mp4, then reuse it instead of running remix again
* #57: API Authorization through API Key can now be enabled. Disabled by default.
* #36: RabbitMQ default credentials are not used anymore. Users can change them to the desired values using .env file.

Bugfixes:
----------

* #137: HLS: missing time adjustments for EXT-X-DATERANGE and EXT-X-PROGRAM-DATE-TIME when not first segment
* #123: Test encrypted sources

v1.11.19
=========

First private beta
