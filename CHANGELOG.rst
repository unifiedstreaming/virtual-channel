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
