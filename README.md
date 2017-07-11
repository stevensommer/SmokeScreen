# SmokeScreen
Host your Plex media with (or without) rclone-crypt on cloud storage, and a local cache for Sonarr and Radarr to scan to prevent Google Drive API limit bans. Special thanks to Reddit users /u/gesis and /u/ryanm91 for most of the heavy lifting.

# Pre-requisites
This project relies on:
* rclone
* unionfs-fuse
* bc (sudo apt-get install bc)
* Plex Media Server
* Plex running as the same user as the scripts (VERY IMPORTANT)
* Sonarr, Radarr, SABnzbd, Torrent Clients (optional)

# Installation
clone the repo then
* move smokescreen.conf to ~/.config/SmokeScreen/smokescreen.conf
* move the scripts to ~/bin/
* make sure all the scripts are executable

# Required rclone Remotes
    
## Without Encryption: ##
Create a remote in rclone that points at the TOP LEVEL of your cloud storage provider. Set the configuration option `$primaryremote` to the remote you created in rclone, and set the configuration option `$cloudsubdir` to a descriptive name. This folder will be created at the top level of your cloud storage automatically when `update.cloud` is run the first time, and media will appear in subfolders beneath it.

## With Encryption ##
Create a remote following the `without encryption` instructions first. Then create a second remote that is a `crypt` remote who's `remote` is the name of your unencrypted remote, followed by `:encrypted`. Follow the instructions to complete setting up the encrypted remote (entering passwords, etc). It is recommended to choose to encrypt filenames. Set the configuration option `$primaryremote` to the encrypted remote you created in rclone, and set the the configuration option `$cloudsubdir` to a descriptive name.

The first time `update.cloud` is run, a folder named `encrypted` will be created at the top level of your cloud storage, and a sub-folder with the encrypted value of `$cloudsubdir` will be created and media will appear in subfolders beneath it.

# Default Configuration Variables

The default configuration creates folders and mount points in your user's home directory. This may not be acceptable for your configuration, so change them to a more suitable location. All steps in this README refer to the `$variable` name and not the `~/path` to avoid confusion.

# Cloud Storage Setup
There is a checking script included that looks for a specific file on cloud storage. Set in the configuration as `$checkfilename`, when Cloud Storage is mounted you should see this file at `$mediadir/$checkfilename`. Use rclone to upload a file of this name to your cloud storage `$cloudsubdir` folder. Example:

    touch ~/google-check
    rclone move ~/google-check GSUITE:Media

Now mount the system by running the `check.mount` script. You should see your cloud storage mounted at `$clouddir` and you should see a union of your local media and cloud storage media directory at `$mediadir`. If you don't, stop here and resolve it before continuing.

# Plex Media Server Configuration
When using SmokeScreen, PMS must be configured to no longer scan for media, and disable media analysis. If you leave these options enabled, you will likely end up with a 24H ban for hitting Google's API too much. Since rclone provides no caching of data from your cloud service, each request hits the API.

Media libraries in Plex must be configured:
* Plex should look at `$plex_shows_folder` for TV Series
* Plex should look at `$plex_movie_folder` for Movies
* Plex should look at `$plex_music_folder` for Music

You must disable:
* Settings -> Library -> Update my library automatically
* Settings -> Library -> Run a partial scan when changes are detected
* Settings -> Library -> Include music libraries in automatic updates
* Settings -> Library -> Update my library periodically
* Settings -> Scheduled Tasks -> Update all libraries during maintenance
* Settings -> Scheduled Tasks -> Upgrade media analysis during maintenance
* Settings -> Scheduled Tasks -> Perform extensive media analysis during maintenance

It is also recommended to disable:
* Settings -> Library -> Empty trash automatically after every scan
* Settings -> Library -> Allow media deletion

If you've created new libraries or modified the paths in existing libraries in Plex, cancel the scans that Plex initiates automatically, as they may cause excessive API usage. We will rescan everything once we're done configuring the other software.

# Sonarr and Radarr Configuration
To use Sonarr/Radarr without the local cache, set the two variables in the configuration `$sonarrenabled` and `$radarrenabled` = 0. Sonarr and Radarr should be configured to use `$mediadir/$media_shows` and `$mediadir/$media_movie` as the root folders for all series/movies. Be warned that excessive scanning by Sonarr/Radarr can lead to 24H bans by Google.

Media that these tools download will follow the following path with the cache disabled:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to $mediadir
* update.cloud will then upload it to cloud storage

To use the local cache, set the two variables in the configuration `$sonarrenabled` and `$radarrenabled` = 1. Reconfigure Sonarr and Radarr to look at `$localcache/$media_shows` and `$localcache/$media_movie` as their root folder. On the 'Connect' tab of the settings pages, add a custom script that points at `sonarr.cache` in Sonarr and `radarr.cache` in Radarr that notifies on `Download` and `Upgrade`.

Media that these tools download will follow the following path with the cache enabled:

* Download client downloads to temp directory
* Sonarr/Radarr "import" the file to $localcache
* The custom script in Sonarr/Radarr will move the file to the $localmedia folder and scan the media in to Plex
* update.cloud will then upload it to cloud storage

If you are NOT using Sonarr or Radarr, set the configuration values for them to 0. Manually add Movies and TV series to `$localmedia/$media_shows` for TV and `$localmedia/$media_movie` for movies. Be sure that media manually placed here follows Plex's media naming expectations.

# Initial scan and cache build
Once all the software is configured correctly, run `scan.media 1` to force a complete scan of all media and build the cache for Sonarr and Radarr if it's enabled. Be prepared to wait, as this will take a while. It might be a good idea to run the scan in a screen so that if your connection is interrupted the scan won't stop: `screen /bin/bash ~/bin/scan.media 1`.

# Automatic Processing
CRON is used to automatically mount the drives, upload content to cloud storage, scan new media in to Plex, keep the cache up to date and remove local copies of media.

Add the following to your user's crontab:

    @hourly /home/USER/bin/update.cloud >> /home/USER/logs/update.cloud.log 2>&1
    0 1 * * * /home/USER/bin/scan.media >> /home/USER/logs/scan.media.log 2>&1
    @hourly /home/USER/bin/nuke.local >> /home/USER/logs/nuke.local.log 2>&1
    */2 * * * * /home/USER/bin/check.mount >> /home/USER/logs/check.mount.log 2>&1

# A Note About Music
Since cloud storage-hosted music doesn't work well with Plex, but we've disabled all automatic scanning, newly added music will no longer appear automatically in Plex. The configuration variables `$plex_music_folder` and `$plex_music_category` are available so that `scan.media` will scan newly added music. Leave these variables blank if you do not use Plex for music.

# Utility Script
There is a script included called pms. This script just sets up the environment so that you can call the Plex media scanner easily. Usage is:

    ./pms "whatever you want to send to the media scanner"
    
Make SURE to wrap the command line with quotation marks, so that the entire thing is treated as argument one being passed in. You can use it to get your library IDs by calling it like this:

    ./pms "--list"
    
The SmokeScreen.conf file must be in place (and properly configured for your environment) before it will work.

# Switching to plexdrive
plexdrive can be used for mounting instead of rclone. rclone is still used to upload new media (and encrypt if you so choose) so configure SmokeScreen to use rclone following the instructions for with or without encryption first, then continue here if you'd like to switch to plexdrive. Download plexdrive and place the binary somewhere (I recommend `/usr/bin/plexdrive`). Configure it to connect to your Google Drive, and then continue below.

## Without Encryption ##
In `smokescreen.conf` add the path to the plexdrive binary at the end of the file:

    plexdrivebin=/usr/bin/plexdrive

In `mount.remote` find:

    ${rclonebin} mount ${rclonemountopts} ${primaryremote}: "${clouddir}" &
    
and replace it with:

    screen -dmS plexdrive ${plexdrivebin} -v 3 "${clouddir}"

## With Encryption ##
In `smokescreen.conf` add the path to the plexdrive binary at the end of the file, and new variables for where plexdrive will mount your media and the name of the rclone remote you'll need to create:

    plexdrivebin=/usr/bin/plexdrive
    plexdrivemount=${HOME}/plexdrive
    plexdriverclone=reverseencryption:
    
Create a new remote in rclone of type `crypt` and give it the name of the value for `$plexdriverclone` set above. Tell it the location is the value of `$plexdrivemount` followed by `/encrypted`. Finish creating the remote using the same encryption settings as the rclone remote created using the 'with encryption' steps for originally configuring SmokeScreen. Once complete, continue with the next steps below.
    
In `mount.remote` find:

    ${rclonebin} mount ${rclonemountopts} ${primaryremote}: "${clouddir}" &
    if [ ! "${cloudsubdir}" = "" ]; then
	    while [ ! -d "${clouddir}/${cloudsubdir}" ]; do
	    	sleep 10
	    done
    fi
    
And replace it with:

    screen -dmS plexdrive ${plexdrivebin} --clear-chunk-max-size=25G -v 3 "${plexdrivemount}"
	while [ ! -d "${plexdrivemount}/encrypted" ]; do
		sleep 10
    done
    ${rclonebin} mount ${rclonemountopts} ${reverseencryption}: "${clouddir}" &
    
In `unmount.remote` add a section to also unmount the plexdrive mount:

    if mountpoint -q $plexdrivemount; then
        echo "$(date "+%d.%m.%Y %T") INFO: Unmounting ${plexdrivemount}"
        fusermount -uz $plexdrivemount 2>&1
    fi

## Sonarr/Radarr And Plex Settings ##
using plexdrive negates the need for caching. Disable the cache by setting both `$sonarrenabled` and `$radarrenabled` in `smokescreen.conf` to 0. Then, reconfigure Sonarr and Radarr to use `$clouddir` as their root instead of `$localcache` and disable the custom post processing scripts `sonarr.cache` and `radarr.cache` as they will no longer work. Add a connection to your Plex server in Sonarr and Radarr to update your library when new content is downloaded. Automatic scanning in Plex can also be re-enabled.
