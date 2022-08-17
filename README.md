# What is Virtual Channel?
A service to create/modify/delete live channels based on existing VOD content or Live
streams, using a playlist-driven workflow, accessible through a REST API.

A channel is created by submitting a SMIL playlist, in which you specify the channel
start date/time and its content, that can reference either Live streams or (stiched
together) VOD assets. SCTE-35 markers can be added to VOD playlists to signal Ad
opportunities downstream.

The service includes a Unified Origin instance for channel playout that, by default,
indefinitely loops through the submitted playlist.

Under a few conditions, it is possible to have a channel switch and play a different
playlist, at a given date/time, by means of a so-called "transition". Virtual Channel
takes care of making the transition seamless for Dash and HLS players.

Transitions can switch from/to any combination of VOD2Live and "pure" Live sources.
Refer to [Virtual Channel's solutions page](https://www.unified-streaming.com/solutions/virtual-channels)
for an overview of the supported scenarios.

## Preliminary settings and limitations
A valid license with VOD2Live permission is required. If you don't have one and want to
evaluate Virtual Channel you can create an account at
https://www.unified-streaming.com/get-started to get a trial license.


Its content must be available in the `UspLicenseKey` environmental variable. In the
shell you will use to launch virtual channel, you should run

```
export UspLicenseKey = <content of your license file>
```

Authentication for S3 storage is managed similarly, using environmental variables
to handle S3 access key, secret key and region. If you plan to use SMIL playlists
referencing a cloud storage, you must set the following:

```
export S3_ACCESS_KEY=<your s3 access key>
export S3_SECRET_KEY=<your s3 secret key>
export S3_REGION=<the s3 region of your storage bucket>
export S3_REMOTE_STORAGE_URL=<the s3 http bucket url>
```

Notice that Virtual Channel can only ever work with one S3 bucket at a time, which is
set at startup time. This means that *any* VOD content referenced in your playlists
must belong to the same S3 bucket (i.e. it is not possible to transition across two
playlists referencing content on two or more different S3 buckets).

:construction_worker: :construction: **To be improved** transitions have the
same (or more stringent) limitations encountered when mixing VOD content in the same
SMIL for Remix. Codec and bitrate must match, number and kind of tracks need to match
as well.

## How to run
First, clone this repository.

Virtual Channel is based on `docker compose`. As such, in order to run it you must make
sure that the `docker compose` command is available in your system. In case you need it,
refer to the [Docker official docs](https://docs.docker.com/compose/install/).

- :arrow_right: **Arm support**. Virtual channel is based on multi-architecture x86_64/arm64
  Alpine docker images. This means that, i.e. on an M1-based Mac, images for your native
  architecture will be used, with no emulation layers.

You can start the Virtual Channel service by running:

```
docker compose up -d
```

If your service is up and running correctly, performing a `curl http://localhost:8000`
should return an `I'm alive` message.

## Documentation and Usage
If your service is up and running correctly, you will find an extensive online
documentation at [`http://localhost:8000/docs`](http://localhost:8000/docs). It will
provide you with an exhaustive list of the available endpoints and it will serve you
as reference to create your API calls and define your workflow.

## Channel workflow walkthrough

### Step 1: create a channel
In general, the Virtual Channel workflow starts with the creation of a channel using the
`PUT /channel/{channel}` endpoint. Two things are needed to generate such request:

- you must choose a channel name and use it to form the URL you will `PUT` to (i.e. 
  create a `rock_concert` event by PUTting to `http://localhost:8000/channel/rock_concert`
  )
- an appropriate SMIL playlist must be provided as an `application/xml`request body
  (see [SMIL creation](SMIL-creation) for instructions on SMIL creation).

The name you have chosen for your channel will reflect in the playout URL: i.e. for the
previous example's `rock_concert` name, the playout URL will be available at
`http://localhost/rock_concert.isml`.

The creation of a live channel can be a long-running operation. For this reason, it's
important to understand that its actual creation is performed by a job that Virtual
Channel schedules for background execution after returning from the PUT operation.

In other words, a PUT operation does not create a new channel immediately and does not
guarantee that the channel will be effectively created without errors. Instead, a
variable amount of time (depending by various factors, such as playlist complexity
and network speed) is required before the channel creation jobs terminates.

In order to understand if your channel is live, you have to check your channel creation
status.

### Step 2: check your channel creation status

As a user, you can monitor at any time the status of your channel by using the
`GET /channel/{channel}/status` endpoint. The possible statuses are:

- **Pending**: your channel is scheduled to be created soon. At this time, no
  modifications to the channel are allowed.
- **In Progress**: a job to create your channel is still running. Additional details are
  provided by the status endpoint. At this time, no modifications to the channel are
  allowed.
- **Success**: the job was successful and your channel is live and accessible at the
  relative playout URL.
- **Failed**: the channel was not created because of an error. In this case, refer to
  the API endpoints allowing you to retrieve the task's logs, which include details on
  the outcome of the `remix` and `mp4split` commands.

When your channel has reached the **Success** status, it will be available for playout
at the corresponding `origin_url` provided by the status endpoint.

If the status is **Pending** or **In Progress**, just wait until the system has finished
the job execution.

If the status is **Failed**, it means something went wrong at playlist processing
time and you should check the remix or mp4split logs using the dedicated endpoints.

### Step 3 (optional): amend/modify an existing channel

If the channel's status is either "Success" or "Failure", performing another `PUT`
operation (usually with a modified SMIL playlist) will overwrite the existing channel.

:arrow_right: this is not the endpoint allowing channel "transitions"! This is mostly
dedicated to "overwrite and forget" channels that have been created with a certain
playlist but have not yet started playing. In fact, attempting a `PUT` on a channel
that has started playout already will just fail.

### Step 4 : list and delete your channels

You can get a list of the existing channels with the `/channel` endpoint.

To remove a channel, perform a `DELETE` operation. This will be allowed only if the
channel creation job is in either one of the **Success** or **Failed** statuses.

:warning: the `DELETE` operation does not perform any check and will successfully delete
also channels that are playing already.


## Transition workflow walkthrough

Once you have created at least one channel, you can proceed to add transitions.

### Step 1: create a transition

It is possible to "transition" an existing channel from its current playlist to another
at a given date/time by means of the dedicated endpoint
`PUT /channel/{channel}/{transition}`. The operation is similar to what is required to
create a channel and it requires two things:

- knowing the name of the channel you want to "transition" and the
  time you want the transition to happen. This information is used to form
  the URL to `PUT` to (i.e. to transition the `rock_concert` event to another
  playlist at `2022-07-18T20:00:00Z`, you should `PUT` to 
  `http://localhost:8000/channel/rock_concert/2022-07-18T20:00:00Z`
  )
- an appropriate SMIL playlist, provided as an `application/xml`request body, describing
  what the channel should transition to. (the same SMIL instructions
  [SMIL creation](SMIL-creation) apply).

:warning: always use transition times expressed in UTC, using one of the two allowed
  time formats `2022-04-14T06:00:00Z` or `2022-04-14T06:00:00.000Z`

As it happens for channels, creating a transition may be a long-running operation and
as such is executed in the background. For this reason, you should always check the
transition creation status after using this endpoint.

### Step 2: check your transition creation status

As a user, you can monitor at any time the status of your transition creation job
by using the `GET /channel/{channel}/{transition}/status` endpoint. The possible
statuses are:

- **Pending**: your transition is scheduled to be created soon. At this time, no
  modifications to the channel or the transition are allowed.
- **In Progress**: a job to create your transition is still running. Additional details are
  provided by the status endpoint. At this time, no modifications to the transition are
  allowed.
- **Success**: the job was successful and your transition is active. At the scheduled time
  the channel will switch to the provided playlist.
- **Failed**: the transition was not created because of an error. In this case, refer to
  the API endpoints allowing you to retrieve the task's logs, which include details on
  the outcome of the `remix` and `mp4split` commands. No modifications have been done
  to the channel.

If the status is **Pending** or **In Progress**, just wait until the system has finished
the job execution.

If the status is **Failed**, it means something went wrong at playlist processing
time and you should check the remix or mp4split logs using the dedicated endpoints.

### Step 3 (optional): amend/modify an existing transition

If the transition's status is either "Success" or "Failure", performing another `PUT`
operation (usually with a modified SMIL playlist) will overwrite the existing transition.

:warning: this endpoint is dedicated to modify/overwrite transitions that have been
scheduled but are not active yet (i.e. their transition time is in the future). If the
transition is active (i.e. transition time is in the past), the endpoint will fail
and will not perform any modifications to the channel. If you want to modify the
playlist that is currently playing in the channel, the right thing to do is just
to create another transition at some point in the future.

### Step 4 : list and delete your transitions

You can get a list of the existing transitions on a given channels with the
`/{channel}/transitions` endpoint.

To remove a transition, perform a `DELETE` operation. This will be allowed only if the
transition creation job is in either one of the **Success** or **Failed** statuses.

:warning: the `DELETE` operation does not perform any check and will successfully delete
also active transitions (i.e. with transition time in the past). In this case, any
device/player that is actively reproducing content related to the playlist being
deleted will stop playback and hang. Hoewer, the underlying channel will still be
operational and will just "revert" to the content of the base channel or of any
transition prior to the one that was deleted.


## SMIL creation

General recommendations on how to create SMIL playlists are available at our doc pages 
[VOD2Live](https://docs.unified-streaming.com/documentation/remix/vod2live/playlist_creation.html)
and [SMIL playlists](https://docs.unified-streaming.com/documentation/remix/avod/smil.html#smil-playlist).

In addition to the above recommendations, Virtual Channel requires you to know a
few more details for the two different cases you may need:

- SMIL playlist to define VOD2Live content and SCTE-35 events
- SMIL playlist pointing to a Live stream

### VOD2Live SMIL

For a VOD2Live playlist, an
`<head>` section with two mandatory options, `vod2live` and `vod2live_start_time`
is required.

Here is an example:

```
<?xml version='1.0' encoding='UTF-8'?>
<smil xmlns="http://www.w3.org/2001/SMIL20/Language">
    <head>
        <meta name="vod2live"
            content="true" />
        <meta name="vod2live_start_time"
            content="2022-04-14T06:00:00Z" />
    </head>
    <body>
        <seq>
            <par>
                <video src="http://usp-s3-storage.s3.eu-central-1.amazonaws.com/tears-of-steel/tears-of-steel-64k.mp4" />
                <video src="http://usp-s3-storage.s3.eu-central-1.amazonaws.com/tears-of-steel/tears-of-steel-av1-1000k.cmfv"/>
            </par>
        </seq>
    </body>
</smil>
```

- :warning: You can use the provided example for your tests, because it is based on
  media stored on a publicly accessible s3 bucket. As such though, it
  will only work if you have *not* set any S3 authentication environmental variables.

You can set a desired channel start time by respecting the ISO-8601 datetime format and
expressing time as UTC. Make sure to keep the value of vod2live to `true`.

You can find additional examples of valid SMIL playlists in this repository's
`test/smils/valid` folder.

### Live SMIL

:construction_worker: :construction: provide instructions and a working example

## Limitations
At the moment, the same limitations apply for Virtual Channel than what mentioned
for [VOD2Live](https://docs.unified-streaming.com/documentation/remix/vod2live/limitations.html)

## FAQ

### Channel/transition creation ###

<details>
  <summary><b>How long will it take, from the moment I perform a PUT operation, for my
channel/transition to be available for playout?</b></summary>
  
The creation of a live channel/transition involves two basic steps: 
- running Unified Remix to process the SMIL playlist and create a remixed mp4
- running mp4split to create an `.isml` server manifest based on the remixed mp4.

This process can take a variable amount of time, depending mainly from two factors:
- complexity of the playlist (playlist length, number of entries, media complexity all
  contribute to Unified Remix processing time)
- I/O speed. For a typical case when media are on a cloud storage, the network
  throughput to the remote storage is a key factor.

This means you can expect from a few seconds (for very simple playlists) to a few
minutes (for long playlists, with many elements) of delay in the availability of your
channel/transition, from the moment you perform a PUT operation to create it.
</details>

<details>
  <summary><b>If the channel/transition creation job needs a variable amount of time, how can I
  proceed to be guaranteed that my channel/transition will be available for playout at a certain
  date/time?</b></summary>
  
The correct procedure is to create your channel/transition well ahead of time, but scheduling playout
at a specific point in time. The recommended steps are the following:

- When creating channels, set the `vod2live_start_time` option to the point in time when
  you need your channel to be available for playout (do not forget to express it as an
  UTC time!).
- When creating transitions, set the transition time to the point in time when
  you need your transition to become active (do not forget to express it as an
  UTC time!).
- Perform your PUT operation well ahead of time to make sure Virtual Channel
  has the time to perform the channel/transition creation operation.

If the channel has been set up and a request is made to Origin before the specified
start time, the client receives a 404 and a log message (i.e.
`FMP4_404 VOD2Live starts at 2021-03-22T21:00:00Z`) is written to the Apache error log.

When the specified start time is reached, the Origin will just start serving requests.
</details>

<details>
  <summary><b>What happens if I specify a start time which is a few seconds in the
  future or in the past?</b></summary>
  
If a channel is created successfully only after its specified "vod2live_start_time",
the channel will work correctly but the Origin will just start serving a live edge
which is "into the channel" already.

The same concept applies to transitions: if Virtual Channel has been able to create
a transition only after its transition time, viewers on the Live edge will experience
the transition later than expected. Notice that other viewers using the "catch-up"
or "restart" functionality will experience a transition at the correct moment in
the presentation.
</details>

<details>
  <summary><b>What should I pay attention to when providing a start time
  (vod2live_start_time option)for my channel?</b></summary>
  
- convert your date/time from local time to UTC
- format your time string in one of the two supported ISO8601 formats (with or without
  decimal digits for seconds):
  - `2021-03-22T21:00:00Z`
  - `2021-03-22T21:00:00.000000Z`
</details>

<details>
  <summary><b>What should I pay attention to when providing a transition time
  for the transition endpoint URLs?</b></summary>
  
The recommendations are identical to the ones for the vod2live_start_time option, that is:

- convert your date/time from local time to UTC
- format your time string in one of the two supported ISO8601 formats (with or without
  decimal digits for seconds):
  - `2021-03-22T21:00:00Z`
  - `2021-03-22T21:00:00.000000Z`
</details>

<details>
  <summary><b>What if I make a mistake in my SMIL when creating a channel/transition?</b></summary>
  
Just wait for your job status to reach the `Success` or `Failed` state and then repeat
the PUT operation with a modified SMIL.

If the channel/transition is already active instead, you should DELETE your channel/transition
and create a new one.
</details>

### Channel deletion/modification ###

<details>
  <summary><b>When can I modify an existing channel/transition?</b></summary>
  
Updates are allowed specifically if all the following conditions are
satisfied:

- the channel/transition creation job has terminated, either successfully or with a failure (i.e.
  the status is either `Failed` or `Success`).
- in case of `Success`, the scheduled channel/transition start time hasn't been reached
  yet. This check can be overridden by specifying a `force` query parameter. Use with
  extreme caution.
</details>

<details>
  <summary><b>When can I delete a channel/transition?</b></summary>
  
Deletes are allowed specifically if all the following conditions are
satisfied:

- the channel/transition creation job has terminated, either successfully or with a failure (i.e.
  the status is either `Failed` or `Success`).

Notice that no checks are performed: if the channel is live and you delete it, you will
break playout.

If you delete a transition, any device currently playing its content will hang but the channel
will keep working, using the content from previous transitions.

This is better clarified by a couple of example scenarios:

```
  |---------------------|-------------|---------->
  ch1                  tr1           tr2
                                                ^
  Viewer 1:                               <-----|>
                                         ^
  Viewer 2:                        <-----|>
                              ^
  Viewer 3:             <-----|>
```

The above is a sketch of a channel with two transitions, with viewers
reproducing it at different point in time. In angular brackets, their
DVR window is represented. Viewer 1 is on the Live edge,
while Viewer 2 and 3 are using the "restart/catch-up" functionality.

Example 1: Transition 2 is deleted
From the deletion moment on, Viewer 1 and 2 will be affected and their
players will hang. Viewer3 will not be affected at all. Any other
viewer joining the stream at this point will experience a channel
only having tr1:
```
   |---------------------|------------------------>
  ch1                   tr1           
```

Example 2: Transition 1 is deleted
From the deletion moment on, Viewer 2 and 3 will be affected.
Viewer 3's player will hang. Viewer's 2 player *may* hang, depending on
how much it still relies on content belonging to transition 1's playlist.
Viewer 1 will not be affected at all. Any other viewer joining the stream
at this point will experience a channel only having tr2:

```
  |-----------------------------------|---------->
  ch1                                tr2
```


</details>

<details>
  <summary><b>What happens when I delete a channel?</b></summary>
  
When you DELETE a channel, the `.isml` server manifest and the corresponding remixed mp4
are deleted. Virtual Channel deletes all the details about the channel (original SMIL,
isml, logs and status details) and the Origin immediately stops serving the content.

If you need to keep an history of deleted channels, you should save all the details you
need before deleting it from Virtual Channels.
</details>

<details>
  <summary><b>What happens when I delete a transition?</b></summary>
  
When you DELETE a transition, if it involved VOD2Live content, the `.isml` server
manifest and the corresponding remixed mp4 are deleted. Virtual Channel deletes all
the details about the transition (original SMIL, isml, logs and status details).

If you need to keep an history of deleted transitions, you should save all the details you
need before deleting it from Virtual Channels.
</details>

<details>
  <summary><b>How should I update an existing channel/transition?</b></summary>
  
Just perform a PUT operation on the same channel/transition endpoint, with a
different SMIL payload.

Remember that changes are not permitted on channels/transitions whose vod2live_start_time
or transition time are in the past, unless you specify the `force`  query parameter
(which is highly discouraged since it will break running streams).
</details>

<details>
  <summary><b>Can I update a channel if that is live already?</b></summary>
  
No, Virtual Channels will prevent you from doing so. If you have made a mistake and you 
want to complitely obliterate the existing channel, you can perform a DELETE and a
subsequent PUT to recreate the channel. Notice though that this will inevitably break playout.

If you just want to switch the existing channel to new content, you should perform a
transition. Notice though that viewers will still be able to access the channel content
_previous_ to the transition by using the "restart/catch-up" functionality or just
scrubbing back.

If you positively want to update a channel which is live already and break running
streams, then you can force the operation by specifying the `force` query parameter.
</details>

<details>
  <summary><b>Can I update a channel/transition if that is not yet active?</b></summary>
  
Yes, if the specified start time has not yet been reached, you can modify the channel/transition
(e.g. provide a new SMIL playlist) by just performing another PUT operation. The
existing channel/transition will be completely overwritten.
</details>

### Troubleshooting ###

| Channel Endpoints               | Status Code | Meaning | How to fix |
|------------------------|-------------|---------|------------|
| PUT /channel/{channel}     | 415         | Wrong body mime type. | Please provide an `application/xml` Content-Type header. |
| PUT /channel/{channel}     | 422         | Your SMIL has a syntax error. | Check your XML syntax      |
| PUT /channel/{channel}     | 400         | Your SMIL is not valid. | Check the options in the head section. Check your body section 
| PUT /channel/{channel}     | 400         | a channel with that name is Live already | Add a transition or use the force query parameter (running streams will break!!!). |
| PUT /channel/{channel}     | 503         | There may be a running job already for the same channel | Wait for the job to reach the `Success` or `Failed` status and retry      |
| DELETE /channel/{channel}     | 404         | You are trying to delete a non-existing channel. | Check the channel name |
| DELETE /channel/{channel}     | 503         | a channel creation/update job is running. Cannot delete right now. | Wait for the job to reach the `Success` or `Failed` status and retry |
| GET /channel/{channel}/status     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}/smil     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}/isml     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}/log/remix     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}/log/mp4split     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}/log     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/{channel}     | 404         | The specified channel could not be found | Double-check the channel name |
| GET /channel/     | 404         | No channels have ever been created or they all have been deleted | Create a new channel |

| Transition Endpoints               | Status Code | Meaning | How to fix |
|------------------------|-------------|---------|------------|
| PUT /channel/{channel}/{transition}     | 415         | Wrong body mime type. | Please provide an `application/xml` Content-Type header. |
| PUT /channel/{channel}/{transition}     | 422         | Your SMIL has a syntax error. | Check your XML syntax      |
| PUT /channel/{channel}/{transition}     | 400         | Your SMIL is not valid. | Check the options in the head section. Check your body section 
| PUT /channel/{channel}/{transition}     | 400         | a transition at the same time is Live already | Create a new transition instead or use the force query parameter (running streams will break!!!). |
| PUT /channel/{channel}/{transition}     | 503         | There may be a running job already for the same transition | Wait for the job to reach the `Success` or `Failed` status and retry      |
| DELETE /channel/{channel}/{transition}     | 404         | You are trying to delete a non-existing transition. | Check the transition name |
| DELETE /channel/{channel}/{transition}     | 503         | a transition creation/update job is running. Cannot delete right now. | Wait for the job to reach the `Success` or `Failed` status and retry |
| GET /channel/{channel}/{transition}/status     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/{transition}/smil     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/{transition}/isml     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/{transition}/log/remix     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/{transition}/log/mp4split     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/{transition}/log     | 404         | The specified transition could not be found | Double-check the transition name |
| GET /channel/{channel}/transitions     | 404         | The specified transition could not be found | Double-check the transition name |
