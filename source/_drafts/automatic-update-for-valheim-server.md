---
title: Automatic Update for Valheim Server
tags: 
- Docker
- Valheim
- Steam
- Dedicated Server
- Bash
---

## Introduction

This guides explains how to set up automatic update for a dedicated Valheim server with Docker.  Automatic update is exciting because it minimizes server downtime as a human no longer needs to manually update and restart the server, giving a true 24/7 server uptime experience.    

This guide assumes you have already successfully set up a dedicated server.  It builds on the previous guide [Host Valheim with Docker]() that covers the basics of running a dedicated server at home.  I recommend reading through the previous guide if you haven't already, as this guide will reference many ideas and keywords introduced previously.  

If you're quite familiar with Valheim servers, Docker, and Bash, feel free to jump straight to using the image I built:

* (link to docker hub)
* (link to github)

## Why automatic update

The idea behind automatic update is simple: if there's a newer version of the Valheim server, upgrade the current server to it.  Updating the Valheim server is necessary due to Valheim's client-server model: whenever the game gets new content, the servers also get updates.  Further, updates to the game are pushed unilaterally--players have no choice whether to update or not.  If there is a version mismatch between a game client and a server, players cannot join the server!  This means there is no way around updating the server.  

The "easy" way to do the update is have the human admin be notified when an update is available.  The human admin then needs to shut down the server, perform the update, and then restart the server.  These are all manual steps that have a single point of failure: the human admin needs to be available.  Imagine the admin is sick, loses internet connection, or otherwise is unavailable for a longer period of time; there is the potential for unacceptable downtime for a 24/7 server.  

Automatic update further reduces manual administration of the server since no human needs to be involved, allowing it to be closer to a true 24/7 uptime.  Valheim is still in its infancy and we can continue to expect a stream of new content and bug fixes that automatic update will handle without friction.  

In summary:

* New game clients cannot join an outdated server.  To keep the server running 24/7 requires updating it as soon as new content is available.   
* Automatic update removes the need for a human administrator to detect the update, stop the server, update the server, and restart it.  This can enable the server to run 24/7 on the latest updates without having to wait for an admin to perform the update.  

## The challenge of automatic update

OK, so if you've gotten this far you may be sold.  Automatic update sounds fantastic!  Unfortunately, the Valheim server itself comes with no automatic update feature.  That means it is up to us to come up with a system that can perform all the steps for automatic update.  This adds additional complexity to our dedicated server software, making it harder to understand, debug, and maintain.  

Another issue is that updating requires stopping the current server.  This can lead to a slew of different problems for players such as:

* How can players be warned that the server is updating (and therefore that they'll be kicked off)?
* Should players or an admin be able to delay or stop an update?  Imagine players are still on the old client (a new Valheim update was sent out while they are still playing).  The old server will work fine for these players until they update their game client.  
* How can players be told the update is complete and the server is back online so they can continue playing instead of manually checking the server list? 

To keep this automatic update guide simple, we won't cover how to handle the challenges associated with notifying players about updates and any impact it could have on an ongoing game.  Notifications about server status are very important and merit a separate guide.  

## Steps for automatic update

The idea behind automatic update is simple: update the server whenever a new version is available.  This can be broken down into several steps:

1.  Detect that new updates are available and that the server is outdated
1.  Stop the current running server so updates can be performed
1.  Perform the actual update and validate the server is updated
1.  Start up the server again

Each of these steps is simple to understand but the implementation of each may not be straightforward.  We will cover each step and see how they can be orchestrated together to build a system for automatic update.   

## Environment for testing

When setting out to build an automatic update system, it is key verify that it behaves as expected before use on an actual server.  Automatic update requires stopping the server, updating it, and starting it up again.  If any of these operations fail, the server will be unusable until an admin is available to correct the issue.  Thus, it is critical to verify these steps work before adding the automatic update feature to a real Valheim server.  This is often how real world software is tested in order to minimize impact customer impact when adding new features.  

Verifying that automatic update works is tricky, (1) since we cannot predict when a new update is coming in (2) and a fresh download of the Valheim server with Steamcmd will always yield the latest version.  To verify that our automatic update works, we can use an outdated version of the Valheim server.  If our code is written correctly, then (1) we expect the server to update to the latest version and (2) we are able to play on the updated server.    
For testing purposes, I'll be using an outdated copy of the Valheim server (version 0.146.8).  The image is available here: [sethmachineio/valheim-server:0.146.8](https://hub.docker.com/layers/139144633/sethmachineio/valheim-server/0.146.8/images/sha256-ec5e3c4ed64175c6f6ade8fc4762a8a110d1e2bce31fc7453ada65b78b2c12ce?context=explore).  

### Logging

The demands of automatic update require orchestrating several different steps: detecting an update, shutting down the server, updating the server, and starting it backup.  In the previous [guide]() we built a simple system: startup the server once and shutdown the server once.  Our new automatic update system can potentially start up, update, and shut down an infinite number of times.  To handle this massive increase in complexity, we will want keep track of the state of automatic update to understand how it is behaving as it runs for real, which will help in debugging any unforeseen issues going forward.  Enter logging, which is exactly what is sounds: keep track of key state about our system as it performs different steps as human readable text.  Previously we had used just `echo` statements, but these require heavy modification to be of use in our new, more complex system (e.g. timestamps, which function the message came from, etc.).  

For logging we will use [b-log](https://github.com/idelsink/b-log) which runs out of the box with logging levels like `INFO`, `WARN`, and `ERROR`.  

To view these logging statements from a running Docker container, use `docker logs <container-name>`, e.g. `docker logs valheim-server`.  It is also possible to save the output to a log file for additional inspection.   

### Set up outdated server

Here is how we can setup an environment to test automatic update:

1.  Pull down the outdated Valheim server docker image: `docker pull sethmachineio/valheim-server:0.146.8`
1.  Run the Docker image as a named container.  You can make up parameters as we're only interested in getting the old Valheim server copy:

    ```bash
    docker run --name=old-valheim -d \
    -p 2456:2456/udp -p 2457:2457/udp -p 2458:2458/udp \
    -v /Users/sdworman/valheim-data:/home/steam/valheim-data \
    --env VALHEIM_SERVER_NAME="sethmachine" \
    --env VALHEIM_WORLD_NAME="MadeUpWorld" \
    --env VALHEIM_PASSWORD="HardToGuessPassword" \
    sethmachineio/valheim-server:0.146.8
    ```
1.  Make a `dev` directory in order to separate files used only local development: `mkdir dev/` 
1.  Copy all of the Valheim server contents from the running image to your local machine.  

    ```bash
    docker cp -r old-valheim:/home/steam/valheim-server/ dev/
    ```
1.  The old server files should now be copied over.  Verify with `ls dev/valheim-server` on your local machine:
    
    ```bash
    % ls dev/valheim-server
    LinuxPlayer_s.debug                     linux64                                 start_server_xterm.sh                   valheim_server.x86_64
    UnityPlayer.so                          server_exit.drp                         steam_appid.txt                         valheim_server_Data
    UnityPlayer_s.debug                     start-valheim-server.sh                 steamapps
    Valheim Dedicated Server Manual.pdf     start_server.sh                         steamclient.so 
    ```
1.  Stop and delete the old container: `docker stop old-valheim && docker rm old-valheim`
1.  Write a new Dockerfile that copies over the old server files instead of downloading it from Steam.  This looks nearly identical to the [Dockerfile from the previous guide](https://github.com/sethmachine/valheim-server-docker/blob/main/Dockerfile).  
    
    ```dockerfile
    FROM cm2network/steamcmd:latest

    # need to be ROOT to install new programs
    USER ROOT

    RUN apt-get update && apt-get install git -y
    
    # where Steam is installed
    ENV STEAM_DIR = "/home/steam/Steam"
    # where steamcmd is installed
    ENV STEAM_CMD_DIR = "/home/steam/steamcmd"
    # where the Valheim server is installed to
    ENV VALHEIM_SERVER_DIR "/home/steam/valheim-server"

    # clone the latest b-log from git, remove git to reduce image size, then allow the Steam user to use b-log
    RUN cd ${STEAM_DIR} && git clone https://github.com/idelsink/b-log.git && apt-get remove git -y && chown -R steam:steam b-log/
    
    USER steam

    # install the Valheim server
    COPY --chown=steam dev/valheim-server ${VALHEIM_SERVER_DIR}
    # RUN ./steamcmd.sh +login anonymous \
    # +force_install_dir $VALHEIM_SERVER_DIR \
    # +app_update 896660 \
    # validate +exit
    
    # where world data is stored, map this to the host directory where your worlds are stored
    # e.g. docker run -v /path/to/host/directory:/home/steam/valheim-data
    ENV VALHEIM_DATA_DIR "/home/steam/valheim-data"
    # don't change the port unless you know what you are doing
    ENV VALHEIM_PORT 2456
    # server and world name are truncated after 1st white space
    # you must set values to the server and world name otherwise the container will exit immediately
    ENV VALHEIM_SERVER_NAME=""
    ENV VALHEIM_WORLD_NAME=""
    ENV VALHEIM_PASSWORD "password"
    
    # the server needs these 3 ports exposed by default
    EXPOSE 2456/udp
    EXPOSE 2457/udp
    EXPOSE 2458/udp
    
    VOLUME ${VALHEIM_DATA_DIR}
    
    # copy over server start script
    COPY --chown=steam start-valheim-server.sh ${VALHEIM_SERVER_DIR}
    WORKDIR ${VALHEIM_SERVER_DIR}
    
    ENTRYPOINT ["./start-valheim-server.sh"]
    ```
    
    Note the "install" command is now just copying over the old Valheim server: `COPY --chown=steam dev/valheim-server ${VALHEIM_SERVER_DIR}`.  A copy of start script can be found here: [start-valheim-server.sh](https://github.com/sethmachine/valheim-server-docker/blob/main/start-valheim-server.sh). 
1.  Create a new test image as needed: `docker build -t old-valheim-server/old-valheim-server -f Dockerfile --no-cache .`.  `Dockerfile` is the name of the Dockerfile in the current directory.  

We will be using this Dockerfile as a basis for writing automatic update and will frequently rebuild it as we add and test additional functionality to support automatic update.  
 
 ## Detect server updates 
 ### Find local build ID
 
 Now we have a way to run an outdated Valheim server, we need to find a way to know there is an update available.  Unfortunately, the Valheim server does not provide any native functionality to do this.  Instead, we will need to examine the server's build ID.  Steam apps (e.g. the Valheim server) indicate version through a [buildid](https://partner.steamgames.com/doc/store/application/builds) (build ID).  We can view all the build IDs of the Valheim server here: [Valheim Server builds](https://steamdb.info/app/896660/patchnotes/).  As of April 19, 2021, the latest build ID is 6508109.  As the Valheim server is a Steam app, we can find the build ID by examining its local app manifest file.  
 
 The manifest can be found here in the Valheim server files: `valheim-server/steamapps/appmanifest_896660.acf`.  Within the Docker container, the full path is `/home/steam/valheim-server/steamapps/appmanifest_896660.acf`.  A description of the format can be found here: [ACF Format](https://github.com/leovp/steamfiles/blob/master/docs/acf_overview.rst). Here is what the file looks like: 
 
 ```bash
 % cat dev/valheim-server/steamapps/appmanifest_896660.acf 
 "AppState"
 {
         "appid"         "896660"
         "Universe"              "1"
         "name"          "Valheim Dedicated Server"
         "StateFlags"            "4"
         "installdir"            "Valheim dedicated server"
         "LastUpdated"           "1616289051"
         "UpdateResult"          "0"
         "SizeOnDisk"            "1051599543"
         "buildid"               "6315977"
         "LastOwner"             "76561198275811401"
         "BytesToDownload"               "569743664"
         "BytesDownloaded"               "569743664"
         "BytesToStage"          "1051599543"
         "BytesStaged"           "1051599543"
         "AutoUpdateBehavior"            "0"
         "AllowOtherDownloadsWhileRunning"               "0"
         "ScheduledAutoUpdate"           "0"
         "InstalledDepots"
         {
                 "1006"
                 {
                         "manifest"              "6688153055340488873"
                         "size"          "59862244"
                 }
                 "896661"
                 {
                         "manifest"              "6588021550109601388"
                         "size"          "991737299"
                 }
         }
         "UserConfig"
         {
         }
 } 
 ```
 
 Of note is the line `"buildid"               "6315977"` which tells us the build ID of the *local* Valheim server.  *Local* here means the copy of the Valheim server we have downloaded already (as opposed to the latest or *remote* version).   Build ID [6315977](https://steamdb.info/patchnotes/6315977/) means we're about 3 builds behind, the latest being [6508109](https://steamdb.info/patchnotes/6508109/) (as of April 19th, 2021).  
 
 We will extract the build ID using a text processing tool called [pcregrep](https://www.pcre.org/).  pcregrep allows writing regular expressions that can match patterns that span multiple lines.  The multiline functionality will be needed later for finding the remote/latest build ID.  Alternative approaches include: (1) Python libaries to parse the ACF format and (2) native Linux [grep](https://www.gnu.org/software/grep/manual/grep.html).  We won't use Python in order to minimize our dependency footprint (it can add nearly 1 GB to the Docker image size).  grep is not an option as it does not support multiline pattern matching which will be needed soon.  
 
 Now to get the local build ID programmatically:
 
1.  Modify the Dockerfile so the first 3 lines look like:
 
     ```dockerfile
        FROM cm2network/steamcmd:latest 
        
        USER root
        
        RUN apt-get update && apt-get install pcregrep -y && apt-get install git -y
    ```

1.  Re-build the Docker image and run it as a named container, e.g. `valheim`.  
1.  Enter the running container using `docker exec -it valheim /bin/bash`.
1.  Enter the directory that contains the manifest, e.g. `cd /home/steam/valheim-server/steamapps`
1.  Verify the app manifests exists, e.g. `cat appmanifest_896660.acf`.  
1.  Pipe output to `pcregrep` and look for "buildid": `cat appmanifest_896660.acf | pcregrep "buildid"`:
    ```bash
            "buildid"               "6315977"
    ```
1.  The line with the build ID is now returned.  We only want the numbers, so we need to modify the regular expression further: `cat appmanifest_896660.acf | pcregrep -o1 -M '"buildid".*"([0-9]+)"'`.  This should output "6315977"!  

The regular expression essentially says: find the line with build ID and then give me back all the numbers surrounded by quotes.  Don't worry if the regular expression looks like voodoo; what is important is that now we have a reliable way to get the local build ID!  

Now we can reliably find the build ID of our local Valheim server.  If we can find the the latest (remote) build ID and compare this to our local build ID, we can know whether the server needs to be updated (i.e. the build IDs do not match).  

### Find remote build ID

Fortunately, Steamcmd provides a few methods to query Steam about the latest build IDs of a Steam app.  `+app_info_update` provides the most recent "app info", which will contain the latest build ID for the Valheim server.  `+app_info_print` will dump this information to standard output (stdout) from which the remote build ID can be extracted.  For more details on the command line options, see [Steamcmd Command Line Options](https://developer.valvesoftware.com/wiki/Command_Line_Options#SteamCMD).  

Within the Docker container, the following command should return the latest info for the Valheim server: `/bin/bash /home/steam/steamcmd/steamcmd.sh +login anonymous +app_info_update 1 +app_info_print 896660 +quit > valheim-app-info`.  Note I've saved the output to a local file for further examination.  Recall that 896600 is the Steam app ID for the Valheim server.  You should see the following output, which is quite verbose (I cut off the parts about Steam updating):

```bash
"896660"
{
        "common"
        {
                "name"          "Valheim Dedicated Server"
                "type"          "Tool"
                "parent"                "892970"
                "oslist"                "windows,linux"
                "osarch"                ""
                "icon"          "1aab0586723c8578c7990ced7d443568649d0df2"
                "logo"          "233d73a1c963515ee4a9b59507bc093d85a4e2dc"
                "logo_small"            "233d73a1c963515ee4a9b59507bc093d85a4e2dc_thumb"
                "clienticon"            "c55a6b50b170ac6ed56cf90521273c30dccb5f12"
                "clienttga"             "35e067b9efc8d03a9f1cdfb087fac4b970a48daf"
                "ReleaseState"          "released"
                "associations"
                {
                }
                "gameid"                "896660"
        }
        "config"
        {
                "installdir"            "Valheim dedicated server"
                "launch"
                {
                        "0"
                        {
                                "executable"            "start_server_xterm.sh"
                                "type"          "server"
                                "config"
                                {
                                        "oslist"                "linux"
                                }
                        }
                        "1"
                        {
                                "executable"            "start_headless_server.bat"
                                "type"          "server"
                                "config"
                                {
                                        "oslist"                "windows"
                                }
                        }
                }
        }
        "depots"
        {
                "1004"
                {
                        "name"          "Steamworks SDK Redist (WIN32)"
                        "config"
                        {
                                "oslist"                "windows"
                        }
                        "manifests"
                        {
                                "public"                "6473168357831043306"
                        }
                        "maxsize"               "39546856"
                        "depotfromapp"          "1007"
                }
                "1005"
                {
                        "name"          "Steamworks SDK Redist (OSX32)"
                        "config"
                        {
                                "oslist"                "macos"
                        }
                        "manifests"
                        {
                                "public"                "2135359612286175146"
                        }
                        "depotfromapp"          "1007"
                }
                "1006"
                {
                        "name"          "Steamworks SDK Redist (LINUX32)"
                        "config"
                        {
                                "oslist"                "linux"
                        }
                        "manifests"
                        {
                                "public"                "6688153055340488873"
                        }
                        "maxsize"               "59862244"
                        "depotfromapp"          "1007"
                }
                "896661"
                {
                        "name"          "Valheim dedicated server Linux"
                        "config"
                        {
                                "oslist"                "linux"
                        }
                        "manifests"
                        {
                                "public"                "7912728285894485280"
                        }
                        "maxsize"               "991429535"
                        "encryptedmanifests"
                        {
                                "experimental"
                                {
                                        "encrypted_gid_2"               "6D9E1220765544EA3B7DC02D088CDD6F"
                                        "encrypted_size_2"              "4F7078B30DF0F9538945B47354BD7DE9"
                                }
                                "unstable"
                                {
                                        "encrypted_gid_2"               "BA8AACD688214C6BA994553BA96CB84F"
                                        "encrypted_size_2"              "E0DC3506113023BDCFD913C511E06C1F"
                                }
                        }
                }
                "896662"
                {
                        "name"          "Valheim dedicated server Windows"
                        "config"
                        {
                                "oslist"                "windows"
                        }
                        "manifests"
                        {
                                "public"                "2464500753503578968"
                        }
                        "maxsize"               "983808497"
                        "encryptedmanifests"
                        {
                                "experimental"
                                {
                                        "encrypted_gid_2"               "293E21DD740054EB79AEE3F34C8D9A48"
                                        "encrypted_size_2"              "C37E100649B0C392D6D76A141F3BA994"
                                }
                                "unstable"
                                {
                                        "encrypted_gid_2"               "C7C4577FDF7CDED20056D854E77C11E8"
                                        "encrypted_size_2"              "8BD953AE12BCF2C3472F0725D682E037"
                                }
                        }
                }
                "branches"
                {
                        "public"
                        {
                                "buildid"               "6287700"
                                "timeupdated"           "1614242525"
                        }
                        "experimental"
                        {
                                "buildid"               "6294553"
                                "description"           "Experimental version of Valheim"
                                "pwdrequired"           "1"
                                "timeupdated"           "1614271566"
                        }
                        "unstable"
                        {
                                "buildid"               "6295625"
                                "description"           "Unstable test version of valheim"
                                "pwdrequired"           "1"
                                "timeupdated"           "1614285732"
                        }
                }
        }
}
``` 

Of interest is the line starting with "branches" followed by "public".  The "buildid" under "public" should contain the latest build ID:

```bash
                "branches"
                {
                        "public"
                        {
                                "buildid"               "6287700"
                                "timeupdated"           "1614242525"
                        }
```

In this case it contains an even more outdated build ID [6287700](https://steamdb.info/patchnotes/6287700/).  Running `cat valheim-app-info | grep "6508109"` doesn't seem to yield anything either (6508109 is the latest build ID as of April 19th, 2021).  

It turns out Steam caches the app info of all installed apps.  When Steamcmd requests updated app info, it will first look at the local cache.  If the cache doesn't exist, only then will it ask Steam for the latest app info.  In the Docker container, the cached app info (for all Steam apps installed, not just Valheim) can be found at `/home/steam/Steam/appcache/appinfo.vdf`.  Let's delete `appinfo.vdf` and then re-run the command:

```bash
rm /home/steam/Steam/appcache/appinfo.vdf
/home/steam/steamcmd/steamcmd.sh +login anonymous +app_info_update 1 +app_info_print 896660 +quit > valheim-app-info 
```

Now look for the latest build ID: `cat valheim-app-info | grep "6508109"`.  This should yield the following output:

```bash
                                "buildid"               "6508109"
                                "buildid"               "6508109"
```

The relevant section of the output now looks like:

```bash
                "branches"
                {
                        "public"
                        {
                                "buildid"               "6508109"
                                "timeupdated"           "1618836460"
                        }

```

To extract the latest build ID we will use regular expressions again.  This time the regular expression will need to span multiple lines (we want the "buildid" inside "public" and not any other "buildid" lines).  This is why pcregrep is being used over vanilla grep; the latter does not support matching across multiple lines).  As an aside, the actual output format is called [VDF (Valve Key Files Format)](https://developer.valvesoftware.com/wiki/KeyValues).  Unfortunately, there is no Linux package to parse this VDF files (though Python libraries do exist, e.g. [ValvePython](https://github.com/ValvePython/vdf)).   

Here is what the regular expression looks like that pulls out the public build ID:

```bash
cat valheim-app-info | pcregrep -o1 -M '"branches".*\n*.*{\n*.*"public".*\n*.*{.*\n*.*"buildid".*"([0-9]+)"'
```

This should output the latest build ID 6508109!  

### Putting it together

It is now possible to extract the local build ID (version of the Valheim server currently being run) and the remote build ID (latest version of the Valheim server).  These two build IDs can be compared to determine if the server has to be updated.  There's a lot to keep track of, so it makes sense to begin organizing our automatic update code into different script files.  Below is what I've come up with for a 1st iteration of checking if the server needs to be updated.  The file is named `update-valheim-server.sh`:

```bash
#!/usr/bin/env bash
# Provides functions to update to a newer Valheim server automatically

# This is where the app manifest is that stores the build ID of the local Valheim server
VALHEIM_SERVER_APP_MANIFEST="$VALHEIM_SERVER_DIR/steamapps/appmanifest_896660.acf"
# This is where the local/current build ID is stored
VALHEIM_SERVER_LOCAL_BUILD_ID=""
# This is what Steam reports is the latest build ID
VALHEIM_SERVER_REMOTE_BUILD_ID=""
# Steam caches all app infos (contain build IDs); need to delete this file to get latest remote build IDs
STEAM_CACHED_APP_INFO="$STEAM_DIR/appcache/appinfo.vdf"

function findAndSetLocalValheimServerBuildId(){
    # Finds the buildId from the Valheim server's local app manifest
    # Check if there was a previous local build ID so we can see if anything changed
    local previousLocalBuildId=""
    if [[ ! -z "${VALHEIM_SERVER_LOCAL_BUILD_ID}" ]]
    then
        local previousLocalBuildId=$VALHEIM_SERVER_LOCAL_BUILD_ID
    fi

    VALHEIM_SERVER_LOCAL_BUILD_ID=$(cat ${VALHEIM_SERVER_APP_MANIFEST} | pcregrep -o1 -M '"buildid".*"([0-9]+)"')

    if [[ ! -z "${previousLocalBuildId}" ]] && [[ ${previousLocalBuildId} != ${VALHEIM_SERVER_LOCAL_BUILD_ID} ]]
    then
        INFO "The local build ID was updated from $previousLocalBuildId to $VALHEIM_SERVER_LOCAL_BUILD_ID"
    else
        INFO "The local build ID is $VALHEIM_SERVER_LOCAL_BUILD_ID"
    fi
}

function findAndSetRemoteValheimServerBuildId(){
    # Finds the latest build ID from Steam
    # Delete the cached app info so the latest build ID is fetched from Steam
    if [[ -f "${STEAM_CACHED_APP_INFO}" ]]
    then
        INFO "Deleting cached app info: $STEAM_CACHED_APP_INFO"
        rm $STEAM_CACHED_APP_INFO
    fi

    # First update the app info get the latest build IDs from the Steam remote
    INFO "Querying the remote server for the latest build ID for the Valheim server"
    local appInfo=$(/bin/bash $STEAMCMD_DIR/steamcmd.sh +login anonymous +app_info_update 1 +app_info_print 896660 +quit)
    # Use a regex that looks for the build ID under the public branch
    # TODO: use a VDF parser, as the regex will easily break if the line order changes
    #                    "branches"
    #                {
    #                        "public"
    #                        {
    #                                "buildid"               "6437354"
    #
    VALHEIM_SERVER_REMOTE_BUILD_ID=$(echo "$appInfo" | pcregrep -o1 -M '"branches".*\n*.*{\n*.*"public".*\n*.*{.*\n*.*"buildid".*"([0-9]+)"')

    INFO "The remote server build ID is $VALHEIM_SERVER_REMOTE_BUILD_ID"
}
```

Note that the script references environment variables defined in both the Dockerfile and the original `start-valheim-server.sh`, and also defines new ones that keep track of local build ID (`VALHEIM_SERVER_LOCAL_BUILD_ID`), remote build ID (`VALHEIM_SERVER_REMOTE_BUILD_ID`), the Valheim server app manifest (`VALHEIM_SERVER_APP_MANIFEST`), and Steam's app info cache (`STEAM_CACHED_APP_INFO`).  
 
## The update loop

In the previous section we were able to find the local build ID and the remote build ID of the Valheim server.  When the local build ID and the remote build ID are not the same, it means our local Valheim server needs to be updated.  Ideally we would want to update as soon as the remote build ID changed.  However, there's no magical way to know that the remote build ID has changed other than by running `findAndSetRemoteValheimServerBuildId` over and over again (as an aside, [SteamWebPipes](findAndSetRemoteValheimServerBuildId) would allow for listening for new builds but it is far too complex to cover here).  This means we will need some mechanism to periodically query for the remote build ID and then compare to the local build ID while the Valheim server is running.  

This is the update loop, which every so often will need to query Steam to find the latest build ID and compare to the build ID of the current server.  The update loop needs to run forever and will only terminate if the Docker container itself is stopped.  In Bash, the loop might start to like like this:

```bash
while true
do
    INFO "Sleeping for $VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY before checking for Valheim server update"
    sleep $VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY
    # Check if the Valheim server needs to be updated, and if so update it
    # Need to stop the Valheim server and then restart it afterwards
    INFO "Checking to see if the Valheim server needs to be updated"
    findAndSetLocalValheimServerBuildId
    findAndSetRemoteValheimServerBuildId
    if [[ "$VALHEIM_SERVER_LOCAL_BUILD_ID" = "$VALHEIM_SERVER_REMOTE_BUILD_ID" ]]
    then
        INFO "The Valheim server is already up to date with build ID $VALHEIM_SERVER_LOCAL_BUILD_ID"
    else
        INFO "Updating the Valheim server from $VALHEIM_SERVER_LOCAL_BUILD_ID (old) to $VALHEIM_SERVER_REMOTE_BUILD_ID (new)"
        shutdownValheimServer
        updateValheimServer
        findAndSetLocalValheimServerBuildId
        assertLocalBuildIsLatest
        startValheimServer
    fi
done
```

Some important details:

* `$VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY` controls how often the update is checked for
* [sleep](https://man7.org/linux/man-pages/man1/sleep.1.html) pauses the process for the specified duration, e.g. `sleep 30s` would sleep for 30 seconds before continuing to execute code.  Note that this means the update loop and the Valheim server must necessarily run in different processes--checking for an update should not impact what the server is doing and vice versa.
* `$VALHEIM_SERVER_LOCAL_BUILD_ID`, `$VALHEIM_SERVER_LOCAL_BUILD_ID` variables keep track of what the local and remote build IDs are at all times
* `shutdownValheimServer`, `updateValheimServer`, `assertLocalBuildIsLatest` and `startValheimServer` are all functions that we will have implement later.  

In regard to `$VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY`, what duration should be chosen? A very short value, e.g. 30 seconds, will waste a lot of resources just to confirm the server is up to date (it is unlikely an update comes out every 30 seconds) and Steam could potentially throttle or block our request.  A longer duration, e.g. 2 hours, risks having the server unusable for up to 2 hours if an update has been pushed out to clients already.  For a heavily used server, a lower value such as 5, 10 or 15 minutes may be best.  The frequency chosen is the absolute maximum amount of time a server would not be joinable from a newer game client.  It is best to use short durations like 5s for locally debugging and confirming that automatic updating (as we are doing in this guide).  The auto update frequency will be an additional parameter that can be customized at runtime when launching the server rather than hardcoded.  

We will now implement the functions from above that we had left undefined.  Where possible, we will abstract each function to its own script file to logically organize the automatic update system.  

### start-valheim-server.sh

This script defines the `startValheimServer` function.  It also creates the `VALHEIM_SERVER_PID` variable that keeps track of the process ID of the server.  The process ID allows to shut down the server without losing data.  

```bash
#!/usr/bin/env bash

# Starts the Valheim server

# Keep track of the Valheim server process ID to shut it down later
VALHEIM_SERVER_PID=""

function startValheimServer()
{
    export templdpath=$LD_LIBRARY_PATH
    export LD_LIBRARY_PATH=./linux64:$LD_LIBRARY_PATH
    export SteamAppId=892970

    INFO "Starting the Valheim server"
    INFO "LD_LIBRARY_PATH: $LD_LIBRARY_PATH"

    INFO "Valheim port is: $VALHEIM_PORT"
    INFO "Valheim server name is: $VALHEIM_SERVER_NAME"
    INFO "Valheim world name is: $VALHEIM_WORLD_NAME"
    if [ "${VALHEIM_SERVER_PUBLIC}" = 0 ]
    then
        WARN "The Valheim server is not set to public.  It will not show up in the server list.  Players must join via IP address instead"
    else
        INFO "The Valheim server is set to public visibility.  It will be visible in the server list.  Players will still need to enter the password to join"
    fi

    cd $VALHEIM_SERVER_DIR
    # start the server as a background process to get its PID ("&" at end of command)
    # "&>>" means append all stdout and stderr to the log file
    ./valheim_server.x86_64 -name $VALHEIM_SERVER_NAME \
    -port $VALHEIM_PORT \
    -world $VALHEIM_WORLD_NAME \
    -password $VALHEIM_PASSWORD \
    -public $VALHEIM_SERVER_PUBLIC \
    -savedir $VALHEIM_DATA_DIR &>> "/home/steam/valheim-data/$VALHEIM_WORLD_NAME-logs.txt" &
    VALHEIM_SERVER_PID=$!
    INFO "Valheim server PID is: $VALHEIM_SERVER_PID"
}
```

Some important details:

* This is more or less a copy paste of `start-valheim-server` script in the previous guide
* the `&` operator at the end a command means to launch that process in the background
* the `$!` operator provides the process ID (pid) of the last process launched in the background (in this case, the Valheim server process)
* `VALHEIM_SERVER_PUBLIC` is a new parameter from recent updates that controls whether the server is visible in the join list.  

### shutdown-valheim-server.sh

This script defines the `shutdownValheimServer` function that shuts down the Valheim server process (not necessarily the Docker container).  Note that it references `VALHEIM_SERVER_PID` from the `start-valheim-server.sh` script.  There is an additional function `shutdownValheimServerAndExit` which shuts down the Valheim server **and** exits the Docker container.  When the server is shut down just for an update, we don't want to exit the container and should use `shutdownValheimServer`.  If the Docker container itself is requested to stop, e.g. `docker stop valheim-server`, then `shutdownValheimServerAndExit` should be run.  

```bash
#!/usr/bin/env bash

# docker sends a SIGTERM and then SIGKILL to the main process
# Valheim needs a SIGINT (CTRL+C) to terminate properly
function shutdownValheimServerAndExit()
{
    WARN "Shutting down the Valheim server.  PID is: $VALHEIM_SERVER_PID"
    # send a SIGINT to shut down the Valheim server gracefully
    kill -2 $VALHEIM_SERVER_PID
    # wait for Valheim to terminate before shutting down the container
    wait $VALHEIM_SERVER_PID
    exit 0
}


function shutdownValheimServer()
{
    # Shut down the Valheim server without exiting the current process
    WARN "Shutting down the Valheim server.  PID is: $VALHEIM_SERVER_PID"
    kill -2 $VALHEIM_SERVER_PID
    wait $VALHEIM_SERVER_PID
}
```

Some important details:

* use `shutdownValheimServerAndExit` when stopping the whole Docker container
* use `shutdownValheimServer` when stopping the Valheim server for an update (and intending to start it again when the update is finished)
* `kill -2` sends a `SIGINT` interrupt signal to the Valheim server (equivalent to CTRL+C), which triggers it to shutdown correctly (no loss of data).  

### update-valheim-server.sh

This is a continuation of update script that we wrote in [Detect Server Updates](#Putting-it-together) which finds the local and remote build IDs.  Let's expand it further to include update loop and the update logic.  

```bash
#!/usr/bin/env bash
# Provides functions to update to a newer Valheim server automatically

# This is where the app manifest is that stores the build ID of the local Valheim server
VALHEIM_SERVER_APP_MANIFEST="$VALHEIM_SERVER_DIR/steamapps/appmanifest_896660.acf"
# This is where the local/current build ID is stored
VALHEIM_SERVER_LOCAL_BUILD_ID=""
# This is what Steam reports is the latest build ID
VALHEIM_SERVER_REMOTE_BUILD_ID=""
# Steam caches all app infos (contain build IDs); need to delete this file to get latest remote build IDs
STEAM_CACHED_APP_INFO="$STEAM_DIR/appcache/appinfo.vdf"
# The PID of the loop that periodically checks whether to update the Valheim server
VALHEIM_SERVER_UPDATE_LOOP_PID=""

function findAndSetLocalValheimServerBuildId(){
    # Finds the buildId from the Valheim server's local app manifest
    # Check if there was a previous local build ID so we can see if anything changed
    local previousLocalBuildId=""
    if [[ ! -z "${VALHEIM_SERVER_LOCAL_BUILD_ID}" ]]
    then
        local previousLocalBuildId=$VALHEIM_SERVER_LOCAL_BUILD_ID
    fi

    VALHEIM_SERVER_LOCAL_BUILD_ID=$(cat ${VALHEIM_SERVER_APP_MANIFEST} | pcregrep -o1 -M '"buildid".*"([0-9]+)"')

    if [[ ! -z "${previousLocalBuildId}" ]] && [[ ${previousLocalBuildId} != ${VALHEIM_SERVER_LOCAL_BUILD_ID} ]]
    then
        INFO "The local build ID was updated from $previousLocalBuildId to $VALHEIM_SERVER_LOCAL_BUILD_ID"
    else
        INFO "The local build ID is $VALHEIM_SERVER_LOCAL_BUILD_ID"
    fi
}

function findAndSetRemoteValheimServerBuildId(){
    # Finds the latest build ID from Steam
    # Delete the cached app info so the latest build ID is fetched from Steam
    if [[ -f "${STEAM_CACHED_APP_INFO}" ]]
    then
        INFO "Deleting cached app info: $STEAM_CACHED_APP_INFO"
        rm $STEAM_CACHED_APP_INFO
    fi

    # First update the app info get the latest build IDs from the Steam remote
    INFO "Querying the remote server for the latest build ID for the Valheim server"
    local appInfo=$(/bin/bash $STEAMCMD_DIR/steamcmd.sh +login anonymous +app_info_update 1 +app_info_print 896660 +quit)
    # Use a regex that looks for the build ID under the public branch
    # TODO: use a VDF parser, as the regex will easily break if the line order changes
    #                    "branches"
    #                {
    #                        "public"
    #                        {
    #                                "buildid"               "6437354"
    #
    VALHEIM_SERVER_REMOTE_BUILD_ID=$(echo "$appInfo" | pcregrep -o1 -M '"branches".*\n*.*{\n*.*"public".*\n*.*{.*\n*.*"buildid".*"([0-9]+)"')

    INFO "The remote server build ID is $VALHEIM_SERVER_REMOTE_BUILD_ID"
}

function assertLocalBuildIsLatest(){
    if [[ ${VALHEIM_SERVER_LOCAL_BUILD_ID} != ${VALHEIM_SERVER_REMOTE_BUILD_ID} ]]
    then
        ERROR "The local build differs from the remote build after updating.  Local build ID: $VALHEIM_SERVER_LOCAL_BUILD_ID.  Remote build ID: $VALHEIM_SERVER_REMOTE_BUILD_ID"
    else
        INFO "The local build ID is the same as the latest remote build ID"
    fi
}

function updateValheimServer(){
    INFO "Updating the Valheim server"
    /bin/bash $STEAMCMD_DIR/steamcmd.sh +login anonymous +force_install_dir $VALHEIM_SERVER_DIR +app_update $VALHEIM_SERVER_APP_ID +quit
}

function updateValheimServerIfNewerBuildExists(){
    # Only call this function before the first time the server ever starts up
    INFO "Checking to see if the Valheim server needs to be updated"
    findAndSetLocalValheimServerBuildId
    findAndSetRemoteValheimServerBuildId
    if [[ "$VALHEIM_SERVER_LOCAL_BUILD_ID" = "$VALHEIM_SERVER_REMOTE_BUILD_ID" ]]
    then
        INFO "The Valheim server is already up to date with build ID $VALHEIM_SERVER_LOCAL_BUILD_ID"
    else
        updateValheimServer
    fi
}

function checkForAndUpdateValheimServer(){
    # Check if the Valheim server needs to be updated, and if so update it
    # Need to stop the Valheim server and then restart it afterwards
    INFO "Checking to see if the Valheim server needs to be updated"
    findAndSetLocalValheimServerBuildId
    findAndSetRemoteValheimServerBuildId
    if [[ "$VALHEIM_SERVER_LOCAL_BUILD_ID" = "$VALHEIM_SERVER_REMOTE_BUILD_ID" ]]
    then
        INFO "The Valheim server is already up to date with build ID $VALHEIM_SERVER_LOCAL_BUILD_ID"
    else
        INFO "Updating the Valheim server from $VALHEIM_SERVER_LOCAL_BUILD_ID (old) to $VALHEIM_SERVER_REMOTE_BUILD_ID (new)"
        shutdownValheimServer
        updateValheimServer
        findAndSetLocalValheimServerBuildId
        assertLocalBuildIsLatest
        startValheimServer
    fi
}

function startUpdateLoop(){
    trap 'shutdownValheimServerAndExit' SIGTERM
    while true
    do
        INFO "Sleeping for $VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY before checking for Valheim server update"
        sleep $VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY &
        sleep_pid=$!
        wait $sleep_pid
        checkForAndUpdateValheimServer
    done
}
```

Some important details:

* `assertLocalBuildIsLatest` checks to see that the local and remote build IDs are the same after an update. If this is not true, then something has gone wrong and this will alert us in an error log statement.  
* `updateValheimServer` runs the Steamcmd command needed to update the server to the latest version.  
* `updateValheimServerIfNewerBuildExists` updates the Valheim server on startup so that whenever a stopped container is resumed (e.g. `docker start valheim-server`) it is updated without going through the update loop.  This function is only meant to run once when the container starts and never again.  
* `checkForAndUpdateValheimServer` gets the latest remote build ID, compares it to the local build ID, and if they are different, performs the update.  This involves stopping the Valheim server (`shutdownValheimServer`), updating it (`updateValheimServer`), and then starting it again (`startValheimServer`).  
* `startUpdateLoop` runs forever and executes `checkForAndUpdateValheimServer` at the frequency dictated by `$VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY`.  If the value were 30m, then the loop would check for an update every 30 minutes.  
* `trap 'shutdownValheimServerAndExit' SIGTERM` intercepts any SIGTERM signals and then requests the Valheim server to stop in order to shut down without data loss (e.g. the container is stopped via `docker stop valheim-server`).  
* `wait $sleep_pid` allows the sleep process to be interrupted by an interrupt signal, i.e. when the Docker container is stopped (`docker stop valheim-server`).  Without this, the sleep would not terminate and the Valheim server would force killed, potentially resulting in loss of data.

### valheim-server-entrypoint.sh

To handle the different runtime parameters and actually kick off the Valheim server and update loop, we will use an entrypoint script.  The entrypoint will also allow a user to disable the automatic update feature if they wish to manually control updates (perhaps running a legacy server, etc.).  

```bash
#!/usr/bin/env bash

# Provides nicely formatting logging
# See: https://github.com/idelsink/b-log
source "$STEAM_DIR/b-log/b-log.sh"
LOG_LEVEL_ALL # All log levels are visible
# The entry point script parses runtime parameters and kicks off the Valheim server
# This script must be executed from the same directory as all the other scripts.
source start-valheim-server.sh
source update-valheim-server.sh
source shutdown-valheim-server.sh

# server name and world name need to be defined at runtime
if [ -z "${VALHEIM_SERVER_NAME}" ]
then
    FATAL "Please set the VALHEIM_SERVER_NAME property.  E.g. use --env VALHEIM_SERVER_NAME=\"My Server Name\""
    exit 1
fi

if [ -z "${VALHEIM_WORLD_NAME}" ]
then
    FATAL "Please set the VALHEIM_WORLD_NAME property.  E.g. use --env VALHEIM_WORLD_NAME=\"AWholeNewWorld\""
    exit 1
fi

if [ "${VALHEIM_SERVER_UPDATE_ON_START_UP}" = 1 ]
then
    INFO "Attempting one time update of the Valheim server on start up"
    updateValheimServerIfNewerBuildExists
fi

function terminateUpdateLoop(){
    INFO "Terminating the update loop.  PID is $VALHEIM_SERVER_UPDATE_LOOP_PID"
    # -15 is equivalent to SIGTERM
    kill -15 $VALHEIM_SERVER_UPDATE_LOOP_PID
    wait $VALHEIM_SERVER_UPDATE_LOOP_PID
    exit 0;
}

if [ "${VALHEIM_SERVER_AUTO_UPDATE}" = 0 ]
then
    # Handle starting and shutting down server normally if no auto update
    # catch Docker's SIGTERM, then send a SIGINT to the Valheim server process
    trap 'shutdownValheimServerAndExit' SIGTERM

    startValheimServer

    # since the server is run in the background, this is needed to keep the main process from exiting
    while wait $VALHEIM_SERVER_PID; [ $? != 0 ]; do true; done
else
    WARN "Experimental auto update is enabled.  The server will automatically update and restart when a new version is detected"
    INFO "Updates to the server will be checked every $VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY"
    
    startValheimServer &
    trap 'terminateUpdateLoop' SIGTERM
    startUpdateLoop &
    VALHEIM_SERVER_UPDATE_LOOP_PID=$!
    INFO "Valheim server update loop PID is: $VALHEIM_SERVER_UPDATE_LOOP_PID"
    while wait $VALHEIM_SERVER_UPDATE_LOOP_PID; [[ "${VALHEIM_SERVER_UPDATE_LOOP_PID}" != 0 ]]; do true; done
fi
```

Some important details:

* the `source` command effectively imports all the variables and functions of another script, so they can be re-used in the entrypoint and all of its child processes.  
* `VALHEIM_SERVER_UPDATE_ON_START_UP` is a runtime parameter that controls whether to update the server whenever starting the container for the first time rather than waiting for the update loop to detect an update.
* `VALHEIM_SERVER_AUTO_UPDATE` enables the auto update feature; if set to 0 then the container will simply run the Valheim server at its current version.  Any other value will enable auto update.  Some users may wish to disable auto update if they are running a legacy server or want full control of updates.  
* `terminateUpdateLoop` handles stopping the update loop if the container is stopped, e.g. `docker stop valheim-server`.  It handles the SIGTERM signal from Docker and then redirects it to update loop process, which causes it to execute `shutdownValheimServerAndExit`.

### Updated Dockerfile

We now need to update the Dockerfile to use these new scripts and create the new parameters that control how automatic update works.  Here is what our updated Dockerfile should look like:

```dockerfile
FROM cm2network/steamcmd:latest

USER root
# Install PCREGREP (http://www.pcre.org/) to extract build IDs from the VDF format
# PCREGREP allows for writing easy to understand regular expressions that can span multiple lines
RUN apt-get update && apt-get install pcregrep -y && apt-get install -y procps && apt-get install git -y

# where Steam is installed
ENV STEAM_DIR "/home/steam/Steam"
# where steamcmd is installed
ENV STEAMCMD_DIR "/home/steam/steamcmd"
# where the Valheim server is installed to
ENV VALHEIM_SERVER_DIR "/home/steam/valheim-server"
# the Steam app ID that uniquely identifies the server
ENV VALHEIM_SERVER_APP_ID 896660
# 1 enables a one time check to update the Valheim server whenever it is first started
ENV VALHEIM_SERVER_UPDATE_ON_START_UP 1
# 1 enables auto update; set to 0 to disable auto update
ENV VALHEIM_SERVER_AUTO_UPDATE 1
# how often to check for server updates
# For format: https://linuxize.com/post/how-to-use-linux-sleep-command-to-pause-a-bash-script/
ENV VALHEIM_SERVER_AUTO_UPDATE_FREQUENCY "30m"

RUN cd ${STEAM_DIR} && git clone https://github.com/idelsink/b-log.git && apt-get remove git -y && chown -R steam:steam b-log/

# changes the uuid and guid to 1000:1000, allowing for the files to save on GNU/Linux
USER steam

# install the Valheim server
COPY --chown=steam dev/valheim-server ${VALHEIM_SERVER_DIR}
#RUN ./steamcmd.sh +login anonymous \
#+force_install_dir $VALHEIM_SERVER_DIR \
#+app_update 896660 \
#validate +exit

# where world data is stored, map this to the host directory where your worlds are stored
# e.g. docker run -v /path/to/host/directory:/home/steam/valheim-data
ENV VALHEIM_DATA_DIR "/home/steam/valheim-data"
# don't change the port unless you know what you are doing
ENV VALHEIM_PORT 2456
# server and world name are truncated after 1st white space
# you must set values to the server and world name otherwise the container will exit immediately
ENV VALHEIM_SERVER_NAME=""
ENV VALHEIM_WORLD_NAME=""
ENV VALHEIM_PASSWORD "password"
# 1 allows viewing the server in the public list; 0 hides it (must join by IP)
ENV VALHEIM_SERVER_PUBLIC 1

# the server needs these 3 ports exposed by default
EXPOSE 2456/udp
EXPOSE 2457/udp
EXPOSE 2458/udp

VOLUME ${VALHEIM_DATA_DIR}

# copy over the scripts to start, update, and shutdown the server
COPY --chown=steam valheim-server-entrypoint.sh ${VALHEIM_SERVER_DIR}
COPY --chown=steam start-valheim-server.sh ${VALHEIM_SERVER_DIR}
COPY --chown=steam update-valheim-server.sh ${VALHEIM_SERVER_DIR}
COPY --chown=steam shutdown-valheim-server.sh ${VALHEIM_SERVER_DIR}

WORKDIR ${VALHEIM_SERVER_DIR}

ENTRYPOINT ["./valheim-server-entrypoint.sh"]
```

Note we will still use the outdated server until we verify the automatic update behaves as expected, as done with `COPY --chown=steam dev/valheim-server ${VALHEIM_SERVER_DIR}`.  

## Testing automatic update

Let's test automatic update under a few different scenarios.  

  




  

       