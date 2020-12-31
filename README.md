# BelchertownWxFeeds
An extension to the Belchertown Skin extension to retrieve weather and other information from a variety of endpoints.

Version 2.0 - the first public release.

BelchertownWxFeeds.py

Copyright 2020-2099 Garry Lockyer - Garry@lockyer.ca
Distributed under terms of the GPLv3

This document is based on similar document for weewx driver WeatherFlowUDP:

"Copyright 2017-2020 Arthur Emerson, vreihen@yahoo.com
Distributed under terms of the GPLv3"

This is an extension for weewx *WITH* the Belchertown skin to get numerous 
weather and travel related feeds.

I believe that I have correctly followed the instructions provided by weewx 
(as implemented for the WeatherFlowUDP driver) in order to provide the complete 
weewx extension package layout here.

*GAL: NEED TO PROVIDE*: https://github.com/weewx/weewx/wiki/extensions#how-to-install-an-extension

Installation should be as simple as grabbing a .ZIP download of this
entire project from the GitHub web interface, and then running this
command:

wee_extension --install BelchertownWxFeeds-master.zip

Worst case, a manual install is simple enough.  At least on my Raspberry Pi, and 
assumming a setup.py install:

1. Copy BelchertownWxFeeds.py from bin/user to /home/weewx/bin/user/BelchertownWxFeeds.py, 

2. Edit /home/weewx/weewx.conf and add the new Beclchertown Extra settings
per the info below.

Please let me (Garry@lockyer.ca) know if I made a mistake with the packaging.  I am not a weewx
expert, and do not claim to be a great programmer or more than a casual user of
GitHub.

-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-

Description:

WxFeeds has two components:

- BelchertownWxFeeds.py and 

- the WxFeeds proxy servers.

Note: BelchertownWxFeeds works with only the weewx Belchertown skin. 

This section is extracted from BelchertownWxFeeds.py:

****** Start of Material From BelchertownWxFeeds.py ******

# This program pulls weather (wx) information and other information 
# from multiple sources (feeds).
#
# It currently supports:
# - ATOM feeds from Environment Canada
#   - weather conditions
#     - forecasts
#     - alerts
#   - marine forecasts
#   - public alerts
#
# - METARs and TAFS from NOAA NWS Aviation Weather Center
# - Forecast, includig hourly forecasts, from NOAA NWS.
# - Alerts from NOAA NWS Storm Prediction Center (land and sea)
# - Hurricane information from NOAA NWS National Hurricane Center
# - Road conditions from Drive BC.
# - Road conditions and other data from 511 Alberta.
#
# WxFeeds also has a proxy server (two actually!) to Authorize 
# access to the proxy server and to:
# - proxy requests to Google Maps API services so that your 
#   Google Maps API key is not in publically accessible web pages.
#   - currently Google Maps' Embedded Maps, Geocoding and Places 
#     are proxied
# - proxy requests to NOAA NWS Forecast Points service, again to 
#   hide your API key.
# - proxy requests to AIR-PORT-CODES (to hide API key and 
#   associated Secret).
#
# WxFeeds supports caching JSON data from Google Maps Geocoding and 
# Places services, NOAA NWS Points service and AIR-PORT-CODES 
# airport information service.  Caching can happen at two levels:
#   - locally (on the same system as WeeWx) using a file based 
#     method and
#   - remotely (via the WxFeeds proxy servers) using redis.
#
#
# See WxFeeds-READ-ME-txt for more information, including 
# installation instructions and information about the WxFeeds 
# proxy server ".env" file.

NOTES:

To Do:

- add more WS DOT traveller information

- clean up Alberta 511

- new NOAA NWS AWS METARs/TAFs

- use Bootstap accordians to reduce length of index.html

***** End of Material From belchertownWxFeeds.py ******

***** belchertownWxFeeds.py ******

Prerequisites:

- weewx
- Belchertown Skin

-various python libraries:

import sys
import io           # For readlines().
import os           # For mkdir()
import datetime
import time
import pytz         # For timezone database.
from pytz import timezone
import requests
import logging
import json         # For loads().

import feedparser   # For parse().
import isodate      # For ISO 8601 duration parsing.
import pathlib
from pathlib import Path

Manual Installation:

1. Extend the Cheetah generator in Belchertown/skin.conf:

Copy bechertownWxFeeds.py to /home/weewx/bin/user.

2. Edit [CheetahGenerator] in skin.conf to include belchertownWxFeeds:

    search_list_extensions = . . .,user.belchertownWxFeeds.WxFeedsSearch

    Note: "search_list_extensions" may be specified in weewx.conf!  Doing so 
          will help keep all changes for Belchertown in weewx.conf 
          and graphs.conf.  As I believe is documented by the developer of 
          Belchertown, putting the information in weewx.conf protects you 
          from changes/upgrades to Belchertown/skin.conf.  Make sure you 
          include ALL extensions including the standard "user.belchertown.getData".

This (the above) stanza does not need to exist in Belchertown/skin.conf.  You can put the 
equivalent information in weewx.conf.  

I use the weewx.conf method and it is the only method documented here.

3. Add [WxFeeds] stanza in Belchertown/skin.conf and/or weewx.conf:

    Similar to above, weewx configuratons can be in skin.conf and/or weewx.conf.  
    I prefer to put as much in weewx.conf as possible and that method is 
    documented below.

4. belchertownWxFeeds uses the weewx logging system, at least to log to syslog 
('cause I haven't implemented or tested a custom logging scheme!).

    Add the following within the [Logging] [[loggers]] stanza: 

    [[[user.belchertownWXFeeds]]]
        level = INFO # CRITICAL, ERROR, WARNING, INFO, DEBUG or NOTSET.
                   # belchertownWxFeeds uses WARNING, INFO and DEBUG.
                   # WARNING should be good for working installations.
                   # Use DEBUG to help troubleshoot setup issues.

5 Edit weewx.conf WxFeeds stanza for belchertownWxFeeds.

6. Optional: Install The WxFeeds Proxy Servers

THIS NEEDS TO BE IMPROVED!

- copy the WxFeeds servers (WxFeedsAuthServer.js and WxFeedsProxyServer.js) from the 
  WxFeeds zip file to a folder on your server.
- copy sample.env from the WxFeeds.zip file to .env in the same folder 
  as the WxFeeds servers.
- use 'systemctl' to start each server and check that they are enabled 
  to automatically restart.

More information based on WxFeeds entries in my weewx.conf:

This is within [StdReport] [[Belchertown]].

    [[[WxFeeds]]]

    # NOTE(s):
    #
    # 1) Yes-No / True-False: Any control that requires a binary value of Yes/No or True/False 
    # can be set using 'yes' or 'true'.  Case is ignored.  Only 'yes' or 'true' will set the 
    # control to 'True'.  Anything else will set the control to 'False'.
    #
    # 2) You can specify 'position optionals' in many places.  'Position optionals' specify:
    #   latitude,
    #   longitude,
    #   zoom and
    #   search_criteria
    #
    #   If 'latitude' and 'longitude' are not specified, 'search_criteria' will 
    #   be used to lookup 'latitude' and 'logitude'.
    #
    # 'Position optionals' are used to set the position of a location or 
    # to set a map's initial zoom level.
    #
    # 3) Reducing Write to SD Card
    #
    #    See 'Minmize writeson SD cards' (see https://github.com/weewx/weewx/wiki/Minimize-writes-on-SD-cards).
    #
    # To reduce writes to the SD Card, I use an in-memory temporary file system (tmpfs) for 
    # weewx output and use a folder within that folder to cache files and data, a folder 
    # structure something like: 
    #
    # /var/weewx/reports        # as HTML_ROOT.
    # /var/weewx/WxFeeds/cache  # as file and data cache.
    #
    # If the cache files are within a normal file system (not an 
    # in memory system such as tmpfs), they will persist across reboots.
    #
    # If the cache files are within an in-memory file system, they 
    # will NOT persist across reboots but will be rebuilt automatically.
    #
    # 4) As part of the effort to reduce write operations, WxFeeds maintains information about 
    #    some files in the local cache folder to determine if they are 'stale' and need to be 
    #    written, or if they are 'not stale' and do not need to be written.
    #
    #    A file is 'stale' if:
    #    - its modified date/time is before weewx.conf's modified date/time.
    #      - if weewx.conf has been modified, files it causes to be created might 
    #        need to be recreated.
    #
    #    - its modified date/time is before BelchertownWxFeeds.py's modified date/time.
    #      - if BelchertownWxFeeds.py has been modified, files it causes to be created 
    #        might need to be created.
    #
    #    - its modified date/time is more recent than the date/time saved (cached) for it.
    #      - a file that has been manually modified might need to be recreated.
    #        - an example of this case is 'custom.css'.  Custom.css is not modified by 
    #          the Belchertown skin but I use it for WxFeeds specific CSS.  If it is manually 
    #          edited, WxFeeds will recreate it with 'my' WxFeed specific CSS. See discussion 
    #         below about 'custom.css'.
    #
    # 5) WxFeeds Caching
    #    Data for endpoints such as Google Maps Places and Maps, AIR-PORT-CODES and NOAA 
    #    NWS can be cached remotely and/or locally.  Yes, you can have a 2-level cache - 
    #    you can cache locally *and* remotely.  A local cache will enhance performance 
    #    for a single local system while a remote cache might enhance pperformance for 
    #    multiple systems.  As said elsewhere, I will be hosting 4 weather stations so 
    #    it's possible that a remote cache serving all the stations will provide some 
    #    benefit to all.
    #
    #    But the real benefit to using a remote cache may be hiding API keys (and 'secrets').
    #    The WxFeeds Proxy server allows you to store sensitive information on the proxy 
    #    server rather than having it 'in the clear' in publically accessible (likely HTML) 
    #    files.
    
        # Folder where information about local files (their modification times) and some data 
        # is saved:

        local_cache_path = "/var/weewx/WxFeeds/cache"

        # WxFeeds Custom CSS Control
        #
        # The Belchertown skin allows "custom" CSS to be specified in Belchertown/custom.css.
        #
        # I used custom.css for WxFeeds specific CSS and go to great lengths to make sure 
        # that that the WxFeeds specific CSS is maintained.  WxFeeds specific CSS in 
        # Beclchertown/custom.css is recreated if custom.css does not exist, when weewx.conf is 
        # updated, when the WxFeeds extension (BelchertownWxFeeds.py) is updated and when 
        # custom.css is edited (if local caching is enabled).
        #
        # While working on the code to keep WxFeeds CSS current, I decided that I shouldn't 
        # force everyone to use my CSS, so I added a control to enable/disable WxFeeds custom CSS, 
        # (so that a user could then actually provide their own CSS for WxFeeds specific HTML).

        disable_WXFeeds_CSS no  # Default: False, set to Yes/True to disable inclusion/maintenance 
                                # of default WxFeeds CSS.

        # Icon Control
        #
        # Icon Base: where icons are stored on the web server that will be serving out index.html.
        #
        # Belchertown stores its images in the Belchertown/images subfolder.
        # BelchertownWxFeeds' install.py copies a few WxFeeds specific icons to Belchertowns' 
        # images folder.
        #
        # If you add images to the Belchertown image folder, they will be FTP'd to 
        # the images subfolder on the server you identified to serve "index.html".
        #
        # If you would like to use a different subfolder FOR WXFEEDS SPECIFIC ICONS, you 
        # can use icon_base to identify that subfolder.  For example, I will be serving four 
        # (4) weather stations from my web server and want them the WxFeeds icons to reside 
        # in a single subfolder.
        # 
        # If icon_base is set (to something other than 'images'), icons must be manually 
        # copied (FTP'd) to icon_base.
        #
        # Possible settings:
        #
        #   icon_base = "https://lockyer.ca/weather/images" # Absolute path.
        #   icon_base = "../images"                         # Relative path.

        # Late Breaking News: I decided to tidy up / organize HTML_ROOT by putting 
        # WxFeeds items in a WxFeeds subfolder so I think it's best to use 
        # an absolute path for 'icon_base'.

        icon_base = "https://www.lockyer.ca/weather/WxFeeds/images"

        # Cache and Proxy Control

        # WxFeeds includes two proxy servers, one for authorization and 
        # the other for proxying various services.  I implemented two 
        # 'cause when I was learning about how to create server based 
        # systems, it was suggested separating authentication and 
        # authorization from actual proxying was a good idea, so . . .
        # Remote proxying is "all or nothing."  All the services that are 
        # proxied will be proxied or none of them will be.

        # Turn proxyng on or off.

        use_wx_feeds_proxy_server = yes

        # URL including port of the WxFeeds proxy servers.
        #
        # You need a user name and password to access these servers!
        #
        # The user name and (hashed) password must be in a ".env" 
        # on the authorization server.

        wx_feeds_authorization_server_url   = "https://rqleo158.hostpapavps.net:51712"
        wx_feeds_proxy_server_url           = "https://rqleo158.hostpapavps.net:61712"

        wx_feeds_proxy_server_username = ""
        wx_feeds_proxy_server_password = ""

        # Google Maps and Other Data Sources Control
        #
        # WxFeeds can access services such as Google Maps, AIR-PORT-CODES and 
        # NOAA's National Weather Service (NWS) to provide services such as 
        # creating interactive maps or to lookup data.

        # Where practical (allowable and possible), each service may take 
        # advantage of WxFeeds' caching capability.
        #
        # Each service that is cached has controls for that service.
        #
        # The controls are only documented under Google Maps Geocoding.

        # Google Maps requires an API key. If you are NOT going 
        # to use WxFeeds proxy servers, you can insert your API 
        # key here and all Google Maps services requests will be 
        # done directly to the Google Maps endpoints.  But!!  Your 
        # API key will be exposed which is not in compliance with 
        # Google Maps recommendatons.

        googlemaps_api_key = ""

        # Google Maps may be produced to show where a weather station or 
        # other point is located.  If a map is created, a clickable map icon 
        # will be displayed to link to the map.  Enable map creation here.

        googlemaps_create_maps = yes

        # Google Maps Geodcoding services may be used to look up 
        # latitude / longitude by an address (which might only be 
        # a city name with or with province / state).  Enable that 
        # feature here.

        googlemaps_geocoding_use = yes

        # The results from a Google Maps Geocoding request may be 
        # cached locally.  Enable that here.

        googlemaps_geocoding_use_local_cache = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        # Set period that cached date will be good for, in minutes.
        #
        # 1 week = 10080 minutes.

        googlemaps_geocoding_cache_expiry_minutes = 10080

        # You can cause the *remote* cache(s) to update by setting the 
        # following control to 'yes'.
        #
        # The ability to force cache misses was put in to aid 
        # development of the WxFeed proxy servers.
        #
        # If you want to force a cache miss locally, delete the 
        # relevent files in 'local_cache_path'.

        googlemaps_geocoding_force_cache_miss = no

        # The Google Maps Places service may be used to retrieve 
        # information about points of interest.  It's used to 
        # look up information about if AIR-PORT-CODES is not 
        # and to lookyo position (latitude and longitude) for 
        # lcoations in many section in WxFeeds.
        #
        # AIR-PORT-CODES seems pretty good at finding information 
        # for actual airports but doesn't handle a station such 
        # as 'CWYY' (a standalone weathee station in Canada).  Google 
        # Maps Places does return information for 'CWYY'.
        #
        # Anyway, Google Maps Places and AIR-PORT-CODES are used 
        # so that you don't have to find and enter latitude and 
        # longitude yourself!
        #
        # And while we're here, Google Maps Geocoding did NOT 
        # return data for 'CWYY', but then Geocoding wants an 
        # address and 'CWYY' is not an address.  I don't think 
        # WxFeeds us GoogleMaps Geocoding any more - just Places.

        googlemaps_places_use = yes

        # Caching controls for Google Maps Places, same as for 
        # Google Geocoding.

        googlemaps_places_use_local_cache = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        googlemaps_places_cache_expiry_minutes = 10080

        googlemaps_places_force_cache_miss = no

        # AIR-PORT-CODES.com is one of many APIs to look up airport 
        # information, including position (latitude and longitude).  
        # I initially used an ICAO API but their service is very 
        # expensive so I switched to AIR-PORT-CODES which seems to 
        # have reasonable pricing.  But!  If you don't use caching 
        # WxFeeds can generate many requests that could cost a lot!
        # For example, with an archive interval of 3 minutes, 20 X 
        # 24 = 480 requests pre day.  In 10 days you could generate 
        # about $20 of requests (at current rates of about 
        # 4400 / 20$).  You've been warned!

        # Control the use of AIR-PORT-CODES here.

        air-port-codes_use = yes

        # AIR-PORT-CODES has rules and controls for using their API.  
        # You can register a URL to issue requests from or if that 
        # doesn't work for you, you may also use a "secret" to 
        # issue requests from any URL.
        #
        # For example, I run weewx on a local system withe WxFeeds 
        # servers on a virtual private server, so two URLs are 
        # involved.  I don't know the URL of my local system (as 
        # AIR-PORT-CODES would see it) and don't want to spend time 
        # figuring that out, so using a secret seems easiest.
        # I currently use the secret from the proxy server but it 
        # has a static IP address so I could probably register it 
        # with AIR-PORT-CODES and only use the API key - a task for 
        # another day!
        #
        # And now that I think about, at this time, you *must* use 
        # a key AND a secret!  I may allow this to be selectable 
        # in the future . . 
        #
        # Anyway, enter the key and secret here.

        air-port-codes_api_key      = ""
        air-port-codes_api_secret   = ""

        # Caching controls for AIR-PORT-CODES, same as above.

        air-port-codes_use_local_cache      = yes
        air-port-codes_cache_expiry_minutes = 10080
        air-port-codes_force_cache_miss     = no

        # NOAA NWS Forecast use a unique X/Y grid system to 
        # identify the location to retrieve a forecast for.
        #
        # Their "points" endpoint allows you to look up the grid 
        # specifiers by location (city and state).
        #
        # If you don't enable this feature you probably have to 
        # go to NOAA NWS to find the grid specifiers but there's 
        # no way to enter them so you pretty much have to use 
        # this feature!

        use_noaa_nws_points                     = yes

        # Caching controls for use_noaa_nws_points, same as above 
        # for other services.

        noaa_nws_points_use_local_cache         = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        noaa_nws_points_cache_expiry_minutes    = 10080
        noaa_nws_points_force_cache_miss        = no

        # Belchertown Include File Control

        # The Belchertown skin provides four (4) places within index.html 
        # that you can insert your own HTML and WxFeeds makes use of that 
        # feature.
        #
        # Further, WxFeed attempts to "place nice" with other extensions 
        # that might use belchertown include files.  You can inform WxFeeds 
        # that other extensions are also using an include file using the 
        # include_file_sequence option:
        #
        # 'only' means that no other extionsion will use the current include 
        # file.  WxFeeds will initialize the file (delete all current 
        # contents) before writing WxFeeds HTML to it.    .
        #
        # 'first' means that this will be the first use of the include file, 
        # implicitly saying that it will be used again by WxWeeds or by another 
        # extension.  WxFeeds will initialize the file before writing WXFeeds 
        # HTML to it.
        #
        # 'subsequent' means that WxFeeds or another extension has used 
        # the include file.  WxFeeds appends HTML to the curren contents.
        #
        # 'last' means that this is the last use of the include.  WxFeeds 
        # appends any HTML it needs to end its usage (currently nothing 
        # extra is written).
        #
        # This is all somewhat concpetual as I know of no other extension 
        # that uses the belchertown include files!  But using them 
        # with WxFeeds seems to work.

        # Options to allow the initialization of include files.
        #
        # Useful to get rid of the contents of an include file that 
        # might be present but no longer needed.  For example, you might 
        # tell WxFeeds to use the "after_forecast" include file but later 
        # decide to use the "after_station_info" file.  If you do nothing 
        # else, the belchertown skin will still use the "after_forecast" 
        # file.  You can use this feature to avoid having to manually delete 
        # an include file you no longer want to use.

        initialize_belchertown_include_file_index_hook_after_station_info   = yes
        #initialize_belchertown_include_file_index_hook_after_forecast       = yes
        #initialize_belchertown_include_file_index_hook_after_snapshot       = yes
        #initialize_belchertown_include_file_index_hook_after_charts         = yes

        # Include file to use now.
        #
        # Yes, only one file should be selected!

        belchertown_include_file = "index_hook_after_station_info.inc"
        #belchertown_include_file = "index_hook_after_forecast.inc"
        #belchertown_include_file = "index_hook_after_snapshot.inc"
        #belchertown_include_file = "index_hook_after_charts.inc"

        # Sequence of current include file.
        #
        # And as above, only one should be selectd.

        include_file_sequence = only
        #include_file_sequence = first
        #include_file_sequence = subsequent
        #include_file_sequence = last

        # Tell WxFeeds to add a "Back To The Top Button" to make it easier 
        # to return to the top.

        add_back_to_top_button = yes

        # Finally, control the content WxFeeds can produce!

        [[[[NOAA NWS - Aviation Weather Center - METAR]]]]

            # "Meteorological Terminal Air Reports" (METAR - current 
            # conditions) and "Terminal Area Forecasts) (TAF) provide 
            # weather informationt tailored for aviators.
            #
            # METARs and TAFs are specified by an international standard 
            # and available from various sites.
            #
            # The NOAA National Weather Serices - Aviation Weather Center 
            # provides METARs and TAFs for airport and weather stations 
            # around the world.  METARs and TAFs can be retrieved from 
            # two endpoints.  I think the one WxFeeds currently uses is 
            # an older method/endpoint for which I have not been able to 
            # find documentation, but was able to use thanks to others' 
            # work.  I intend to move to the newer Text Data Services 
            # endpoint/API, and give user the choice to use either or both.

            # Nav Canada operates the Canadian Air Traffic Control system and 
            # provides weather information to aviators, via telephone and 
            # the Aviation Weather Web Site (AWWS).  Use the following 
            # controls to display links to AWWS and/or US AWS Briefing site.

            display_link_to_navcan_awws = yes / no
            display_link_to_us_awc_briefing = yes / no

            # WxFeeds can retrieve METARs and/or TAFs for multiple stations.  
            # A staton can be an airport or a weather station.
            #
            # Airports and weather stations may have two (or more?) identifiers, 
            # one issued by the International Cival Aviation Organization (ICAO) 
            # and the other issued by the International Air Transport Association 
            # (IATA).  ICAO codes, in my eperience, are used more often than IATA 
            # codes but AIR-PORT-CODES uses IATO codes so WxFeeds is cognizant 
            # of both.
            #
            # In the list of Stations below, you can specify a station by either 
            # code and may need to use both because NOAA NWS AWC requires ICAO 
            # codes but AIR-PORT-CODES uses IATA codes.
            #
            # ICAO codes are four (4) characters in length.
            # IATA codes are three (3) characters in length.
            #
            # The convention to enter both is to separate the codes by a "-".
            #
            # You can enter them in either order, ICAO first then IATA, or IATA 
            # first then ICAO.  Whichever code you enter first will be used as 
            # the station ID presented by WxFeeds.  For example, you might prefer 
            # to refer to "Los Angeles International Airport" as LAX rather 
            # than KLAX, the ICAO code needed to get METARs/TAFs.  In this 
            # case, you can enter [[[[[[[LAX-KLAX]]]]]]: LAX will be used for 
            # AIR-PORT-CODES and display while KLAX will be used to retrieve 
            # METARs/TAFs.
            #
            # If enabled, AIR-PORT-CODES will be used before Google Maps Places 
            # because it's spefically intented to provide airport information 
            # while Google Map Places is more general.
            #
            # If AIR-PORT-CODES is not enabled or the airport code cannot be 
            # found by AIR-PORT-CODES, Google Maps Places will be used, if 
            # enabled.  Google Map Places may return "strange" results for an 
            # airport code.  For example, 'CYKA-YKA' will return information for 
            # 'Jules Cyka Ltd.'.  In such cases, you can enter the location 
            # (airport name) and position (latitude/longitude) manually.
            #
            # You can specify 'position optionals': 'latitude', 'longitude', 
            # 'zoom' and 'search_criteria'.
 
            [[[[[Stations]]]]]

                # Default controls for the list of stations.
                #
                # May be specified individualy for each Station.

                get_metars  = yes     # Retrieve METARs
                get_tafs    = yes     # Retrieve TAFs    
                translate   = yes     # Translate the report from a highly condensed format 
                                      # into a much more user-friendly format.
                past_hours  = 3         # Retrieve information for the provious X hours.

                # The following applies to each Station:
                #
                # A code [[[[[[INHERE]]]]]]] is required, and is the only required control.
                #
                # The ICAO code will be used to request MATARs/TAFs from NOAA NWS AWC.
                # The IATO code will be used to look up information for the Station:
                #
                # If location is not entered, it will be looked up.
                # If position (latitude and longitude) is not entered, latitude and longitude 
                # will be looked up.
                # If location_website_url is not entered, it will be looked up.
                #
                # location and position can be looked up by AIR-PORT-CODE and Google Maps, 
                # but only AIR-PORT-CODES can provide a website for a location.
                #
                # You can also specify station_website_url.
                # location_website_url is associated with the name of a location while 
                # station_website_url is associated with a station code.
                #
                # For the stations below, I allow AIR-PORT-CODES to lookup a 
                # location_website_url but enter a station_website_url.
                #
                # Location_website_urls returned by AIR-PORT-CODES seem to be the airport's 
                # public facing site.
                #
                # I use station_website_url to provide a link to more technical/aviator 
                # focused information provided by SkyVector.

                    [[[[[[CWYY]]]]]]

                        # This station is a weather station, not an airport.
                        # AIR-PORT-CODES will not find it but Google Maps Places will.

                        # latitude / longitude are from Government of Canada Meteorological Station Catalogue:
                        # https://open.canada.ca/data/en/dataset/9764d6c6-3044-450c-ac5a-383cedbfef17
                        # AeroWeather App is different!

                        location = "Osoyoos BC - Automatic"
                        latitude = 49.0283
                        longitude = -119.4411

                        station_website_url = "https://weather.gc.ca/city/pages/bc-69_metric_e.html"

                    [[[[[[CYYF-YYF]]]]]]

                        station_website_url = "https://skyvector.com/airport/CYYF/Penticton-Airport"
                        past_hours = 3

                    [[[[[[CYLW-YLW]]]]]]

                        station_website_url = "https://skyvector.com/airport/CYLW/Kelowna-Airport"

                    [[[[[[CYKA-YKA]]]]]]

                        # Found by AIR-PORT-CODES but not by Google Maps Places.

#                        location = "Kamloops International Airport"
#                        latitude = 50.7022
#                        longitude = -120.4440
#                        station_website_url = "https://skyvector.com/airport/CYKA/Kamloops-Airport"

#                    [[[[[[CYVR-YVR]]]]]]

#                        station_website_url = "https://skyvector.com/airport/CYVR/Vancouver-International-Airport"

#                    [[[[[[CYYC-YYC]]]]]]

#                        station_website_url = "https://skyvector.com/airport/CYYC/Calgary-International-Airport"

#                    [[[[[[LAX-KLAX]]]]]]

#                        station_website_url = "https://skyvector.com/airport/LAX/Los-Angeles-International-Airport"

            [[[[Environment Canada]]]]

            # Environment Canada provides forecasts and advisories via ATOM/RSS feeds.
            #
            # There are three (3) Environment Canada sections:
            #
            #   [[[[[Citys]]]]]    # List of cities to get weather and/or alerts for.
            #
            #   [[[[[Public Alerts]]]]] # List of areas to get public alerts for.
            #
            #   [[[[[Marine Areas]]]]]  # A list of marine areas to get weather and advisories for.
            #
            # You can have multiple Environment Canada stanzas, each of which can have one or more 
            # Environment sections.
            #
            # 'Position optionals' may entered for each City, Public Alert Area or Marine Area.

                [[[[[Citys]]]]]    # List of cities to get weather and/or alerts for.

                # You can select to retrieve and display weather (current conditions and/or 
                # forecast) and/or alerts.
                # You can decide to display alerts before ("first") or after weather.
                #
                # Links to the feeds are towards the bottom of each forecast page.
                # Click on the feed link and then weather, even if you just want alerts.  
                # WxFeeds will use the weather URL to come up with an alerts URL.
                #
                # Copy the URL from the Environment Canada page to your weewx confiuration 
                # file.
                # Locating feed URLs for the other subsections is similar - find the RSS feed 
                # icon on the Environment Canada page of interest and copy it to your 
                # weewx configuration file.

                # Defaults for the following list of citys:
                #
                # May be specified individualy for each City.

                    alerts_first    = yes       # Display alerts first - before weather.
                    inline_forecast = yes       # Display a forecast in weewx - inline with weather.
                    get_weather     = yes       # Display weather and forecasts (if inline_forecast="yes")
                    get_alerts      = no        # Display alerts.

#                    [[[[[[Drumheller, AB]]]]]]

#                        get_alerts = yes
#                        feed_url = "https://weather.gc.ca/rss/city/ab-62_e.xml"

#                    [[[[[[Calgary, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-52_e.xml"

#                    [[[[[[Red Deer, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-29_e.xml"

#                    [[[[[[Rocky Mountain House, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-16_e.xml"

#                    [[[[[[Kindersly, SK]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/sk-21_e.xml"

                    [[[[[[Osoyoos, BC]]]]]]
 
                        feed_url = "https://weather.gc.ca/rss/city/bc-69_e.xml"

                    [[[[[[Penticton, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-84_e.xml"

                    [[[[[[Kelowna, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-48_e.xml"

                    [[[[[[Vernon, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-27_e.xml"

                [[[[[Public Alerts]]]]] # List of areas to get public alerts for.

                    [[[[[[Okanagan Valley]]]]]]

                        feed_url ="https://weather.gc.ca/rss/battleboard/bc30_e.xml"

                    [[[[[[Nicola]]]]]]

                        feed_url = "https://weather.gc.ca/rss/battleboard/bc32_e.xml"

                    [[[[[[Similkameen]]]]]]

                        feed_url ="https://weather.gc.ca/rss/battleboard/bc31_e.xml"

                [[[[[Marine Areas]]]]]  # A list of marine areas to get weather and advisories for.

                # A feed may contain weather and advisories for more than one area so 
                # the area name is used to select the area of interest.  This means you 
                # MUST enter an area name exactly as it is found in the feed, which
                # should be the same as found on the Environment Canada web page.

#                    [[[[[[Queen Charlotte Strait]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/12400_e.xml"

#                    [[[[[[Juan de Fuca Strait - west entrance]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/07000_e.xml"

#                    [[[[[[Juan de Fuca Strait - central strait]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/07000_e.xml"

#                    [[[[[[Strait of Georgia - north of Nanaimo]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/14300_e.xml"

#                    [[[[[[Strait of Georgia - south of Nanaimo]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/14300_e.xml"

#            [[[[Drive BC]]]]

                # Drive BC provides information about road conditions, incidents and other events.

                # You can change where the Drive BC data is displayed by selecting a 
                # Belchertown skin include file:

                #belchertown_include_file = "index_hook_after_station_info.inc"
                #belchertown_include_file = "index_hook_after_forecast.inc"
                #belchertown_include_file = "index_hook_after_snapshot.inc"
#                belchertown_include_file = "index_hook_after_charts.inc"

                # include_file_sequence

#                include_file_sequence = only
                #include_file_sequence = first
                #include_file_sequence = subsequent
                #include_file_sequence = last

#                display_links_to_drive_bc = both # Display a link to Drive BC: top, bottom or both.

                # Map Center.

                # 'Position optionals' may be used here.

#                latitude = "49.1555"
#                longitude = "-119.5262" 
#                zoom = "12"

#                dbc_district_filter         = ""   # Select district: "district | district | district"

#                dbc_route_filter            = ""   # Select route: "rout | route | route"

                #dbc_type_filter             = ""   # Select event type.
                #dbc_type_filter             = "Incident"
                #dbc_type_filter             = "Current Planned"
                #dbc_type_filter             = "Future Planned"
                #dbc_type_filter             = "Road Condition"
#                dbc_type_filter             = "Incident | Current Planned | Road Condition"

#                dbc_severity_filter         = ""   # Select severity.

#                display_event_attributes        = "yes"    # Display event attributes for each event.
#                display_event_attribute_lists   = "yes"    # Display a list of event attributes.
                                                            # The purpose of these control is to display attributes 
                                                            # that can be used in the above filters.

#                [[[[[Areas]]]]]    # List of areas.
                                    # Areas are aggregated together into a single list without duplicates.
                                    # Selecting 'Entire Province' with another area works but is wasteful.

#                    [[[[[[Entire Province]]]]]]
#                    [[[[[[Thompson Okanagan]]]]]]
#                    [[[[[[Kootenays]]]]]]

            [[[[NOAA NWS - Forecasts]]]]

                # NOAA NWS provides feeds for forecasts for the USA.
                #
                # As above, NOAA NWS Forecast uses a unique grid system to retrieve a forecast.
                # NOAA NWS provides a service to translate latitude / longitude to the NOAA 
                # NWS grid system.
                #
                # You can manually enter latitude and longitude or you can allow Google Maps 
                # to look up latitude and longitude using the Google Maps Places service.
                #
                # Once latitude and longitude are known, WxFeeds using the NOAA NWS Points 
                # service to get the NOAA NWS Forecast grid values for the location of interest.
                #
                # <<Station Location>> may be used to get a forecast for your weather station 
                # location, if you have entered latitude and longitude for it.
                #
                # 'Position optionals' may entered for each Location.

                units = us    # Select units = 'us' (imperial) or 'si' (metric).
                              # May be set individually for each Location.

                [[[[[Locations]]]]] # List of Locations to get forecasts for.

                    # These controls set defaults for a list of Locations.
                    #
                    # Each control can be set individually for each Location.

                    get_forecast            = yes   # Get forecast by day - there are usally two entries for each day.
                    get_hourly_forecast     = no    # Get hourly forecast - the maximum number of hours is {need to research!}.
                    hourly_forecast_hours   = 24    # Number of hours to display for hourly forecast.

                    [[[[[[<<Station Location>>]]]]]]

                    [[[[[[Oroville, WA]]]]]]

                        get_forecast		= yes
                        get_hourly_forecast = yes
                        units 				= si

                    [[[[[[Seattle, WA]]]]]]

                        get_forecast 			= yes
                        get_hourly_forecast 	= no
                        hourly_forecast_hours 	= 36

#            [[[[NOAA NWS - Storm Prediction Center]]]]

                # NOAA NWS Storm Prediction Center issues weather alerts for land locations 
                # and marine areas.
                #
                # URLs for alert feeds can be found at https://alerts.weather.gov.
				# Alerts are offered by Zone and/or County.
				# Maps of NOAA zones cane be found at https://www.weather.gov/pimar/PubZone
				# but it looks like not all zones are on the maps.  SoI have not found 
				# a good, reliable way to identify the geographic area for a NOAA zone.  
				# The NOAA feed does have data fields for geographic infortation but 
				# I have not seen a feed with data so don't know if it can be used.
				# Perhaps I'll test for non-empty data and log something if detected.
            
                # NOAA NWS SPC also maintains list of all alerts.  Use this control to 
                # display a link to all current alerts.
				#
				# 'Position optionals' may be entered for each Alert.

#               display_links_to_all_current_alerts = "after"

#                [[[[[Alerts]]]]]    # A list of locations to get alerts for.

#                    [[[[[[Okanogan County]]]]]] # Location name.  This value MUST be unique!
                                                 # If not unique, weewx will not start.
                                                 # There are zones with the same name so I append 
                                                 # the zone ID to create a unique value.

#                        feed_url = "https://alerts.weather.gov/cap/wwaatmget.php?x=WAC047&y=0"

#            [[[[Washington State - Department of Transportation]]]]

                # The Washington State Department of Transportion provides several feeds for 
                # traveller information.
                #
                # Currently, only "border wait times" has been implemented.
                #
                # 'Position optionals' may be used for some services / subjects.

#                belchertown_include_file = "index_hook_after_station_info.inc"
                #belchertown_include_file = "index_hook_after_forecast.inc"
                #belchertown_include_file = "index_hook_after_snapshot.inc"
                #belchertown_include_file = "index_hook_after_charts.inc"

                #include_file_sequence = "only"
                #include_file_sequence = "first"
#                include_file_sequence = "subsequent"
                #include_file_sequence = "last"

#                ws_dot_access_code = "5809abc2-6afb-4ab7-bca1-7be1695b7086"
#                ws_dot_use_local_cache = "no"
#                ws_dot_cache_update_period_minutes = 1000000
#                ws_dot_force_cache_update = "yes"

#                [[[[[Traveler Information]]]]] # List of various traveller information subjects.

#                    [[[[[[Border Crossings]]]]]]

                      # 'Position optionals' may be used to set map zoom.

#            [[[[511 Alberta]]]]

                # Very similar to Drive BC.
                #
                # This section does NOT currently display any maps for any resource and 
                # needs to be enhanced same as Drive BC.

                #belchertown_include_file ="index_hook_after_station_info.inc"
                #belchertown_include_file ="index_hook_after_forecast.inc"
                #belchertown_include_file ="index_hook_after_snapshot.inc"
#                belchertown_include_file ="index_hook_after_charts.inc"

                #include_file_sequence = "only"
                #include_file_sequence = "first"
#                include_file_sequence = "subsequent"
                #include_file_sequence = "last"

#                [[[[[Resources]]]]]

#                    [[[[[[Road Conditions]]]]]]

#                        area_filter = "BANFF | JASPER"

#                    [[[[[[Alerts]]]]]]

#                    [[[[[[Events]]]]]]

#                        event_type_filter = "accidentsAndIncidents"
#                        roadway_name_filter = "HWY-16"

#                    [[[[[[Ferries]]]]]]

#                    [[[[[[Parks]]]]]]

                        #park_name_filter_condition = "equal"
#                        park_name_filter_condition = "contains"

#                        park_name_filter = "Sylvan | Big | Bow"

#                    [[[[[[Cameras]]]]]]

#                        display_disabled_cameras = "no"
#                        roadway_name_filter = "Hwy 1 | Hwy 2"

#                    [[[[[[Grouped Cameras]]]]]]

#                        display_disabled_cameras = "no"
#                        roadway_name_filter = "Hwy 1 | Hwy 2"

#            [[[[NOAA NWS - National Hurricane Center]]]]

                # Hurricane information from NOAA NWS - National Hurricane Center.
                #
                # Information about hurrincanes can be very large!
                #
                # 'Position optionals' are not used.

#                [[[[[Basins]]]]]

#                    [[[[[[Atlantic Basin Tropical Cyclones]]]]]]

#                        feed_url = "https://www.nhc.noaa.gov/index-at.xml"
BelchertownWxFeeds.py

Copyright 2020-2099 Garry Lockyer - Garry@lockyer.ca
Distributed under terms of the GPLv3

This document is based on similar document for weewx driver WeatherFlowUDP:

"Copyright 2017-2020 Arthur Emerson, vreihen@yahoo.com
Distributed under terms of the GPLv3"

This is an extension for weewx *WITH* the Belchertown skin to get numerous 
weather and travel related feeds.

I believe that I have correctly followed the instructions provided by weewx 
(as implemented for the WeatherFlowUDP driver) in order to provide the complete 
weewx extension package layout here.

*GAL: NEED TO PROVIDE*: https://github.com/weewx/weewx/wiki/extensions#how-to-install-an-extension

Installation should be as simple as grabbing a .ZIP download of this
entire project from the GitHub web interface, and then running this
command:

wee_extension --install BelchertownWxFeeds-master.zip

Worst case, a manual install is simple enough.  At least on my Raspberry Pi, and 
assumming a setup.py install:

1. Copy BelchertownWxFeeds.py from bin/user to /home/weewx/bin/user/BelchertownWxFeeds.py, 

2. Edit /home/weewx/weewx.conf and add the new Beclchertown Extra settings
per the info below.

Please let me (Garry@lockyer.ca) know if I made a mistake with the packaging.  I am not a weewx
expert, and do not claim to be a great programmer or more than a casual user of
GitHub.

-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-

Description:

WxFeeds has two components:

- BelchertownWxFeeds.py and 

- the WxFeeds proxy servers.

Note: BelchertownWxFeeds works with only the weewx Belchertown skin. 

This section is extracted from BelchertownWxFeeds.py:

****** Start of Material From BelchertownWxFeeds.py ******

# This program pulls weather (wx) information and other information 
# from multiple sources (feeds).
#
# It currently supports:
# - ATOM feeds from Environment Canada
#   - weather conditions
#     - forecasts
#     - alerts
#   - marine forecasts
#   - public alerts
#
# - METARs and TAFS from NOAA NWS Aviation Weather Center
# - Forecast, includig hourly forecasts, from NOAA NWS.
# - Alerts from NOAA NWS Storm Prediction Center (land and sea)
# - Hurricane information from NOAA NWS National Hurricane Center
# - Road conditions from Drive BC.
# - Road conditions and other data from 511 Alberta.
#
# WxFeeds also has a proxy server (two actually!) to Authorize 
# access to the proxy server and to:
# - proxy requests to Google Maps API services so that your 
#   Google Maps API key is not in publically accessible web pages.
#   - currently Google Maps' Embedded Maps, Geocoding and Places 
#     are proxied
# - proxy requests to NOAA NWS Forecast Points service, again to 
#   hide your API key.
# - proxy requests to AIR-PORT-CODES (to hide API key and 
#   associated Secret).
#
# WxFeeds supports caching JSON data from Google Maps Geocoding and 
# Places services, NOAA NWS Points service and AIR-PORT-CODES 
# airport information service.  Caching can happen at two levels:
#   - locally (on the same system as WeeWx) using a file based 
#     method and
#   - remotely (via the WxFeeds proxy servers) using redis.
#
#
# See WxFeeds-READ-ME-txt for more information, including 
# installation instructions and information about the WxFeeds 
# proxy server ".env" file.

NOTES:

To Do:

- add more WS DOT traveller information

- clean up Alberta 511

- new NOAA NWS AWS METARs/TAFs

- use Bootstap accordians to reduce length of index.html

***** End of Material From belchertownWxFeeds.py ******

***** belchertownWxFeeds.py ******

Prerequisites:

- weewx
- Belchertown Skin

-various python libraries:

import sys
import io           # For readlines().
import os           # For mkdir()
import datetime
import time
import pytz         # For timezone database.
from pytz import timezone
import requests
import logging
import json         # For loads().

import feedparser   # For parse().
import isodate      # For ISO 8601 duration parsing.
import pathlib
from pathlib import Path

Manual Installation:

1. Extend the Cheetah generator in Belchertown/skin.conf:

Copy bechertownWxFeeds.py to /home/weewx/bin/user.

2. Edit [CheetahGenerator] in skin.conf to include belchertownWxFeeds:

    search_list_extensions = . . .,user.belchertownWxFeeds.WxFeedsSearch

    Note: "search_list_extensions" may be specified in weewx.conf!  Doing so 
          will help keep all changes for Belchertown in weewx.conf 
          and graphs.conf.  As I believe is documented by the developer of 
          Belchertown, putting the information in weewx.conf protects you 
          from changes/upgrades to Belchertown/skin.conf.  Make sure you 
          include ALL extensions including the standard "user.belchertown.getData".

This (the above) stanza does not need to exist in Belchertown/skin.conf.  You can put the 
equivalent information in weewx.conf.  

I use the weewx.conf method and it is the only method documented here.

3. Add [WxFeeds] stanza in Belchertown/skin.conf and/or weewx.conf:

    Similar to above, weewx configuratons can be in skin.conf and/or weewx.conf.  
    I prefer to put as much in weewx.conf as possible and that method is 
    documented below.

4. belchertownWxFeeds uses the weewx logging system, at least to log to syslog 
('cause I haven't implemented or tested a custom logging scheme!).

    Add the following within the [Logging] [[loggers]] stanza: 

    [[[user.belchertownWXFeeds]]]
        level = INFO # CRITICAL, ERROR, WARNING, INFO, DEBUG or NOTSET.
                   # belchertownWxFeeds uses WARNING, INFO and DEBUG.
                   # WARNING should be good for working installations.
                   # Use DEBUG to help troubleshoot setup issues.

5 Edit weewx.conf WxFeeds stanza for belchertownWxFeeds.

6. Optional: Install The WxFeeds Proxy Servers

THIS NEEDS TO BE IMPROVED!

- copy the WxFeeds servers (WxFeedsAuthServer.js and WxFeedsProxyServer.js) from the 
  WxFeeds zip file to a folder on your server.
- copy sample.env from the WxFeeds.zip file to .env in the same folder 
  as the WxFeeds servers.
- use 'systemctl' to start each server and check that they are enabled 
  to automatically restart.

More information based on WxFeeds entries in my weewx.conf:

This is within [StdReport] [[Belchertown]].

    [[[WxFeeds]]]

    # NOTE(s):
    #
    # 1) Yes-No / True-False: Any control that requires a binary value of Yes/No or True/False 
    # can be set using 'yes' or 'true'.  Case is ignored.  Only 'yes' or 'true' will set the 
    # control to 'True'.  Anything else will set the control to 'False'.
    #
    # 2) You can specify 'position optionals' in many places.  'Position optionals' specify:
    #   latitude,
    #   longitude,
    #   zoom and
    #   search_criteria
    #
    #   If 'latitude' and 'longitude' are not specified, 'search_criteria' will 
    #   be used to lookup 'latitude' and 'logitude'.
    #
    # 'Position optionals' are used to set the position of a location or 
    # to set a map's initial zoom level.
    #
    # 3) Reducing Write to SD Card
    #
    #    See 'Minmize writeson SD cards' (see https://github.com/weewx/weewx/wiki/Minimize-writes-on-SD-cards).
    #
    # To reduce writes to the SD Card, I use an in-memory temporary file system (tmpfs) for 
    # weewx output and use a folder within that folder to cache files and data, a folder 
    # structure something like: 
    #
    # /var/weewx/reports        # as HTML_ROOT.
    # /var/weewx/WxFeeds/cache  # as file and data cache.
    #
    # If the cache files are within a normal file system (not an 
    # in memory system such as tmpfs), they will persist across reboots.
    #
    # If the cache files are within an in-memory file system, they 
    # will NOT persist across reboots but will be rebuilt automatically.
    #
    # 4) As part of the effort to reduce write operations, WxFeeds maintains information about 
    #    some files in the local cache folder to determine if they are 'stale' and need to be 
    #    written, or if they are 'not stale' and do not need to be written.
    #
    #    A file is 'stale' if:
    #    - its modified date/time is before weewx.conf's modified date/time.
    #      - if weewx.conf has been modified, files it causes to be created might 
    #        need to be recreated.
    #
    #    - its modified date/time is before BelchertownWxFeeds.py's modified date/time.
    #      - if BelchertownWxFeeds.py has been modified, files it causes to be created 
    #        might need to be created.
    #
    #    - its modified date/time is more recent than the date/time saved (cached) for it.
    #      - a file that has been manually modified might need to be recreated.
    #        - an example of this case is 'custom.css'.  Custom.css is not modified by 
    #          the Belchertown skin but I use it for WxFeeds specific CSS.  If it is manually 
    #          edited, WxFeeds will recreate it with 'my' WxFeed specific CSS. See discussion 
    #         below about 'custom.css'.
    #
    # 5) WxFeeds Caching
    #    Data for endpoints such as Google Maps Places and Maps, AIR-PORT-CODES and NOAA 
    #    NWS can be cached remotely and/or locally.  Yes, you can have a 2-level cache - 
    #    you can cache locally *and* remotely.  A local cache will enhance performance 
    #    for a single local system while a remote cache might enhance pperformance for 
    #    multiple systems.  As said elsewhere, I will be hosting 4 weather stations so 
    #    it's possible that a remote cache serving all the stations will provide some 
    #    benefit to all.
    #
    #    But the real benefit to using a remote cache may be hiding API keys (and 'secrets').
    #    The WxFeeds Proxy server allows you to store sensitive information on the proxy 
    #    server rather than having it 'in the clear' in publically accessible (likely HTML) 
    #    files.
    
        # Folder where information about local files (their modification times) and some data 
        # is saved:

        local_cache_path = "/var/weewx/WxFeeds/cache"

        # WxFeeds Custom CSS Control
        #
        # The Belchertown skin allows "custom" CSS to be specified in Belchertown/custom.css.
        #
        # I used custom.css for WxFeeds specific CSS and go to great lengths to make sure 
        # that that the WxFeeds specific CSS is maintained.  WxFeeds specific CSS in 
        # Beclchertown/custom.css is recreated if custom.css does not exist, when weewx.conf is 
        # updated, when the WxFeeds extension (BelchertownWxFeeds.py) is updated and when 
        # custom.css is edited (if local caching is enabled).
        #
        # While working on the code to keep WxFeeds CSS current, I decided that I shouldn't 
        # force everyone to use my CSS, so I added a control to enable/disable WxFeeds custom CSS, 
        # (so that a user could then actually provide their own CSS for WxFeeds specific HTML).

        disable_WXFeeds_CSS no  # Default: False, set to Yes/True to disable inclusion/maintenance 
                                # of default WxFeeds CSS.

        # Icon Control
        #
        # Icon Base: where icons are stored on the web server that will be serving out index.html.
        #
        # Belchertown stores its images in the Belchertown/images subfolder.
        # BelchertownWxFeeds' install.py copies a few WxFeeds specific icons to Belchertowns' 
        # images folder.
        #
        # If you add images to the Belchertown image folder, they will be FTP'd to 
        # the images subfolder on the server you identified to serve "index.html".
        #
        # If you would like to use a different subfolder FOR WXFEEDS SPECIFIC ICONS, you 
        # can use icon_base to identify that subfolder.  For example, I will be serving four 
        # (4) weather stations from my web server and want them the WxFeeds icons to reside 
        # in a single subfolder.
        # 
        # If icon_base is set (to something other than 'images'), icons must be manually 
        # copied (FTP'd) to icon_base.
        #
        # Possible settings:
        #
        #   icon_base = "https://lockyer.ca/weather/images" # Absolute path.
        #   icon_base = "../images"                         # Relative path.

        # Late Breaking News: I decided to tidy up / organize HTML_ROOT by putting 
        # WxFeeds items in a WxFeeds subfolder so I think it's best to use 
        # an absolute path for 'icon_base'.

        icon_base = "https://www.lockyer.ca/weather/WxFeeds/images"

        # Cache and Proxy Control

        # WxFeeds includes two proxy servers, one for authorization and 
        # the other for proxying various services.  I implemented two 
        # 'cause when I was learning about how to create server based 
        # systems, it was suggested separating authentication and 
        # authorization from actual proxying was a good idea, so . . .
        # Remote proxying is "all or nothing."  All the services that are 
        # proxied will be proxied or none of them will be.

        # Turn proxyng on or off.

        use_wx_feeds_proxy_server = yes

        # URL including port of the WxFeeds proxy servers.
        #
        # You need a user name and password to access these servers!
        #
        # The user name and (hashed) password must be in a ".env" 
        # on the authorization server.

        wx_feeds_authorization_server_url   = "https://rqleo158.hostpapavps.net:51712"
        wx_feeds_proxy_server_url           = "https://rqleo158.hostpapavps.net:61712"

        wx_feeds_proxy_server_username = ""
        wx_feeds_proxy_server_password = ""

        # Google Maps and Other Data Sources Control
        #
        # WxFeeds can access services such as Google Maps, AIR-PORT-CODES and 
        # NOAA's National Weather Service (NWS) to provide services such as 
        # creating interactive maps or to lookup data.

        # Where practical (allowable and possible), each service may take 
        # advantage of WxFeeds' caching capability.
        #
        # Each service that is cached has controls for that service.
        #
        # The controls are only documented under Google Maps Geocoding.

        # Google Maps requires an API key. If you are NOT going 
        # to use WxFeeds proxy servers, you can insert your API 
        # key here and all Google Maps services requests will be 
        # done directly to the Google Maps endpoints.  But!!  Your 
        # API key will be exposed which is not in compliance with 
        # Google Maps recommendatons.

        googlemaps_api_key = ""

        # Google Maps may be produced to show where a weather station or 
        # other point is located.  If a map is created, a clickable map icon 
        # will be displayed to link to the map.  Enable map creation here.

        googlemaps_create_maps = yes

        # Google Maps Geodcoding services may be used to look up 
        # latitude / longitude by an address (which might only be 
        # a city name with or with province / state).  Enable that 
        # feature here.

        googlemaps_geocoding_use = yes

        # The results from a Google Maps Geocoding request may be 
        # cached locally.  Enable that here.

        googlemaps_geocoding_use_local_cache = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        # Set period that cached date will be good for, in minutes.
        #
        # 1 week = 10080 minutes.

        googlemaps_geocoding_cache_expiry_minutes = 10080

        # You can cause the *remote* cache(s) to update by setting the 
        # following control to 'yes'.
        #
        # The ability to force cache misses was put in to aid 
        # development of the WxFeed proxy servers.
        #
        # If you want to force a cache miss locally, delete the 
        # relevent files in 'local_cache_path'.

        googlemaps_geocoding_force_cache_miss = no

        # The Google Maps Places service may be used to retrieve 
        # information about points of interest.  It's used to 
        # look up information about if AIR-PORT-CODES is not 
        # and to lookyo position (latitude and longitude) for 
        # lcoations in many section in WxFeeds.
        #
        # AIR-PORT-CODES seems pretty good at finding information 
        # for actual airports but doesn't handle a station such 
        # as 'CWYY' (a standalone weathee station in Canada).  Google 
        # Maps Places does return information for 'CWYY'.
        #
        # Anyway, Google Maps Places and AIR-PORT-CODES are used 
        # so that you don't have to find and enter latitude and 
        # longitude yourself!
        #
        # And while we're here, Google Maps Geocoding did NOT 
        # return data for 'CWYY', but then Geocoding wants an 
        # address and 'CWYY' is not an address.  I don't think 
        # WxFeeds us GoogleMaps Geocoding any more - just Places.

        googlemaps_places_use = yes

        # Caching controls for Google Maps Places, same as for 
        # Google Geocoding.

        googlemaps_places_use_local_cache = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        googlemaps_places_cache_expiry_minutes = 10080

        googlemaps_places_force_cache_miss = no

        # AIR-PORT-CODES.com is one of many APIs to look up airport 
        # information, including position (latitude and longitude).  
        # I initially used an ICAO API but their service is very 
        # expensive so I switched to AIR-PORT-CODES which seems to 
        # have reasonable pricing.  But!  If you don't use caching 
        # WxFeeds can generate many requests that could cost a lot!
        # For example, with an archive interval of 3 minutes, 20 X 
        # 24 = 480 requests pre day.  In 10 days you could generate 
        # about $20 of requests (at current rates of about 
        # 4400 / 20$).  You've been warned!

        # Control the use of AIR-PORT-CODES here.

        air-port-codes_use = yes

        # AIR-PORT-CODES has rules and controls for using their API.  
        # You can register a URL to issue requests from or if that 
        # doesn't work for you, you may also use a "secret" to 
        # issue requests from any URL.
        #
        # For example, I run weewx on a local system withe WxFeeds 
        # servers on a virtual private server, so two URLs are 
        # involved.  I don't know the URL of my local system (as 
        # AIR-PORT-CODES would see it) and don't want to spend time 
        # figuring that out, so using a secret seems easiest.
        # I currently use the secret from the proxy server but it 
        # has a static IP address so I could probably register it 
        # with AIR-PORT-CODES and only use the API key - a task for 
        # another day!
        #
        # And now that I think about, at this time, you *must* use 
        # a key AND a secret!  I may allow this to be selectable 
        # in the future . . 
        #
        # Anyway, enter the key and secret here.

        air-port-codes_api_key      = ""
        air-port-codes_api_secret   = ""

        # Caching controls for AIR-PORT-CODES, same as above.

        air-port-codes_use_local_cache      = yes
        air-port-codes_cache_expiry_minutes = 10080
        air-port-codes_force_cache_miss     = no

        # NOAA NWS Forecast use a unique X/Y grid system to 
        # identify the location to retrieve a forecast for.
        #
        # Their "points" endpoint allows you to look up the grid 
        # specifiers by location (city and state).
        #
        # If you don't enable this feature you probably have to 
        # go to NOAA NWS to find the grid specifiers but there's 
        # no way to enter them so you pretty much have to use 
        # this feature!

        use_noaa_nws_points                     = yes

        # Caching controls for use_noaa_nws_points, same as above 
        # for other services.

        noaa_nws_points_use_local_cache         = yes

        # 60 min / hour * 24 hours / day * 7 days = 10080 minutes.

        noaa_nws_points_cache_expiry_minutes    = 10080
        noaa_nws_points_force_cache_miss        = no

        # Belchertown Include File Control

        # The Belchertown skin provides four (4) places within index.html 
        # that you can insert your own HTML and WxFeeds makes use of that 
        # feature.
        #
        # Further, WxFeed attempts to "place nice" with other extensions 
        # that might use belchertown include files.  You can inform WxFeeds 
        # that other extensions are also using an include file using the 
        # include_file_sequence option:
        #
        # 'only' means that no other extionsion will use the current include 
        # file.  WxFeeds will initialize the file (delete all current 
        # contents) before writing WxFeeds HTML to it.    .
        #
        # 'first' means that this will be the first use of the include file, 
        # implicitly saying that it will be used again by WxWeeds or by another 
        # extension.  WxFeeds will initialize the file before writing WXFeeds 
        # HTML to it.
        #
        # 'subsequent' means that WxFeeds or another extension has used 
        # the include file.  WxFeeds appends HTML to the curren contents.
        #
        # 'last' means that this is the last use of the include.  WxFeeds 
        # appends any HTML it needs to end its usage (currently nothing 
        # extra is written).
        #
        # This is all somewhat concpetual as I know of no other extension 
        # that uses the belchertown include files!  But using them 
        # with WxFeeds seems to work.

        # Options to allow the initialization of include files.
        #
        # Useful to get rid of the contents of an include file that 
        # might be present but no longer needed.  For example, you might 
        # tell WxFeeds to use the "after_forecast" include file but later 
        # decide to use the "after_station_info" file.  If you do nothing 
        # else, the belchertown skin will still use the "after_forecast" 
        # file.  You can use this feature to avoid having to manually delete 
        # an include file you no longer want to use.

        initialize_belchertown_include_file_index_hook_after_station_info   = yes
        #initialize_belchertown_include_file_index_hook_after_forecast       = yes
        #initialize_belchertown_include_file_index_hook_after_snapshot       = yes
        #initialize_belchertown_include_file_index_hook_after_charts         = yes

        # Include file to use now.
        #
        # Yes, only one file should be selected!

        belchertown_include_file = "index_hook_after_station_info.inc"
        #belchertown_include_file = "index_hook_after_forecast.inc"
        #belchertown_include_file = "index_hook_after_snapshot.inc"
        #belchertown_include_file = "index_hook_after_charts.inc"

        # Sequence of current include file.
        #
        # And as above, only one should be selectd.

        include_file_sequence = only
        #include_file_sequence = first
        #include_file_sequence = subsequent
        #include_file_sequence = last

        # Tell WxFeeds to add a "Back To The Top Button" to make it easier 
        # to return to the top.

        add_back_to_top_button = yes

        # Finally, control the content WxFeeds can produce!

        [[[[NOAA NWS - Aviation Weather Center - METAR]]]]

            # "Meteorological Terminal Air Reports" (METAR - current 
            # conditions) and "Terminal Area Forecasts) (TAF) provide 
            # weather informationt tailored for aviators.
            #
            # METARs and TAFs are specified by an international standard 
            # and available from various sites.
            #
            # The NOAA National Weather Serices - Aviation Weather Center 
            # provides METARs and TAFs for airport and weather stations 
            # around the world.  METARs and TAFs can be retrieved from 
            # two endpoints.  I think the one WxFeeds currently uses is 
            # an older method/endpoint for which I have not been able to 
            # find documentation, but was able to use thanks to others' 
            # work.  I intend to move to the newer Text Data Services 
            # endpoint/API, and give user the choice to use either or both.

            # Nav Canada operates the Canadian Air Traffic Control system and 
            # provides weather information to aviators, via telephone and 
            # the Aviation Weather Web Site (AWWS).  Use the following 
            # controls to display links to AWWS and/or US AWS Briefing site.

            display_link_to_navcan_awws = yes / no
            display_link_to_us_awc_briefing = yes / no

            # WxFeeds can retrieve METARs and/or TAFs for multiple stations.  
            # A staton can be an airport or a weather station.
            #
            # Airports and weather stations may have two (or more?) identifiers, 
            # one issued by the International Cival Aviation Organization (ICAO) 
            # and the other issued by the International Air Transport Association 
            # (IATA).  ICAO codes, in my eperience, are used more often than IATA 
            # codes but AIR-PORT-CODES uses IATO codes so WxFeeds is cognizant 
            # of both.
            #
            # In the list of Stations below, you can specify a station by either 
            # code and may need to use both because NOAA NWS AWC requires ICAO 
            # codes but AIR-PORT-CODES uses IATA codes.
            #
            # ICAO codes are four (4) characters in length.
            # IATA codes are three (3) characters in length.
            #
            # The convention to enter both is to separate the codes by a "-".
            #
            # You can enter them in either order, ICAO first then IATA, or IATA 
            # first then ICAO.  Whichever code you enter first will be used as 
            # the station ID presented by WxFeeds.  For example, you might prefer 
            # to refer to "Los Angeles International Airport" as LAX rather 
            # than KLAX, the ICAO code needed to get METARs/TAFs.  In this 
            # case, you can enter [[[[[[[LAX-KLAX]]]]]]: LAX will be used for 
            # AIR-PORT-CODES and display while KLAX will be used to retrieve 
            # METARs/TAFs.
            #
            # If enabled, AIR-PORT-CODES will be used before Google Maps Places 
            # because it's spefically intented to provide airport information 
            # while Google Map Places is more general.
            #
            # If AIR-PORT-CODES is not enabled or the airport code cannot be 
            # found by AIR-PORT-CODES, Google Maps Places will be used, if 
            # enabled.  Google Map Places may return "strange" results for an 
            # airport code.  For example, 'CYKA-YKA' will return information for 
            # 'Jules Cyka Ltd.'.  In such cases, you can enter the location 
            # (airport name) and position (latitude/longitude) manually.
            #
            # You can specify 'position optionals': 'latitude', 'longitude', 
            # 'zoom' and 'search_criteria'.
 
            [[[[[Stations]]]]]

                # Default controls for the list of stations.
                #
                # May be specified individualy for each Station.

                get_metars  = yes     # Retrieve METARs
                get_tafs    = yes     # Retrieve TAFs    
                translate   = yes     # Translate the report from a highly condensed format 
                                      # into a much more user-friendly format.
                past_hours  = 3         # Retrieve information for the provious X hours.

                # The following applies to each Station:
                #
                # A code [[[[[[INHERE]]]]]]] is required, and is the only required control.
                #
                # The ICAO code will be used to request MATARs/TAFs from NOAA NWS AWC.
                # The IATO code will be used to look up information for the Station:
                #
                # If location is not entered, it will be looked up.
                # If position (latitude and longitude) is not entered, latitude and longitude 
                # will be looked up.
                # If location_website_url is not entered, it will be looked up.
                #
                # location and position can be looked up by AIR-PORT-CODE and Google Maps, 
                # but only AIR-PORT-CODES can provide a website for a location.
                #
                # You can also specify station_website_url.
                # location_website_url is associated with the name of a location while 
                # station_website_url is associated with a station code.
                #
                # For the stations below, I allow AIR-PORT-CODES to lookup a 
                # location_website_url but enter a station_website_url.
                #
                # Location_website_urls returned by AIR-PORT-CODES seem to be the airport's 
                # public facing site.
                #
                # I use station_website_url to provide a link to more technical/aviator 
                # focused information provided by SkyVector.

                    [[[[[[CWYY]]]]]]

                        # This station is a weather station, not an airport.
                        # AIR-PORT-CODES will not find it but Google Maps Places will.

                        # latitude / longitude are from Government of Canada Meteorological Station Catalogue:
                        # https://open.canada.ca/data/en/dataset/9764d6c6-3044-450c-ac5a-383cedbfef17
                        # AeroWeather App is different!

                        location = "Osoyoos BC - Automatic"
                        latitude = 49.0283
                        longitude = -119.4411

                        station_website_url = "https://weather.gc.ca/city/pages/bc-69_metric_e.html"

                    [[[[[[CYYF-YYF]]]]]]

                        station_website_url = "https://skyvector.com/airport/CYYF/Penticton-Airport"
                        past_hours = 3

                    [[[[[[CYLW-YLW]]]]]]

                        station_website_url = "https://skyvector.com/airport/CYLW/Kelowna-Airport"

                    [[[[[[CYKA-YKA]]]]]]

                        # Found by AIR-PORT-CODES but not by Google Maps Places.

#                        location = "Kamloops International Airport"
#                        latitude = 50.7022
#                        longitude = -120.4440
#                        station_website_url = "https://skyvector.com/airport/CYKA/Kamloops-Airport"

#                    [[[[[[CYVR-YVR]]]]]]

#                        station_website_url = "https://skyvector.com/airport/CYVR/Vancouver-International-Airport"

#                    [[[[[[CYYC-YYC]]]]]]

#                        station_website_url = "https://skyvector.com/airport/CYYC/Calgary-International-Airport"

#                    [[[[[[LAX-KLAX]]]]]]

#                        station_website_url = "https://skyvector.com/airport/LAX/Los-Angeles-International-Airport"

            [[[[Environment Canada]]]]

            # Environment Canada provides forecasts and advisories via ATOM/RSS feeds.
            #
            # There are three (3) Environment Canada sections:
            #
            #   [[[[[Citys]]]]]    # List of cities to get weather and/or alerts for.
            #
            #   [[[[[Public Alerts]]]]] # List of areas to get public alerts for.
            #
            #   [[[[[Marine Areas]]]]]  # A list of marine areas to get weather and advisories for.
            #
            # You can have multiple Environment Canada stanzas, each of which can have one or more 
            # Environment sections.
            #
            # 'Position optionals' may entered for each City, Public Alert Area or Marine Area.

                [[[[[Citys]]]]]    # List of cities to get weather and/or alerts for.

                # You can select to retrieve and display weather (current conditions and/or 
                # forecast) and/or alerts.
                # You can decide to display alerts before ("first") or after weather.
                #
                # Links to the feeds are towards the bottom of each forecast page.
                # Click on the feed link and then weather, even if you just want alerts.  
                # WxFeeds will use the weather URL to come up with an alerts URL.
                #
                # Copy the URL from the Environment Canada page to your weewx confiuration 
                # file.
                # Locating feed URLs for the other subsections is similar - find the RSS feed 
                # icon on the Environment Canada page of interest and copy it to your 
                # weewx configuration file.

                # Defaults for the following list of citys:
                #
                # May be specified individualy for each City.

                    alerts_first    = yes       # Display alerts first - before weather.
                    inline_forecast = yes       # Display a forecast in weewx - inline with weather.
                    get_weather     = yes       # Display weather and forecasts (if inline_forecast="yes")
                    get_alerts      = no        # Display alerts.

#                    [[[[[[Drumheller, AB]]]]]]

#                        get_alerts = yes
#                        feed_url = "https://weather.gc.ca/rss/city/ab-62_e.xml"

#                    [[[[[[Calgary, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-52_e.xml"

#                    [[[[[[Red Deer, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-29_e.xml"

#                    [[[[[[Rocky Mountain House, AB]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/ab-16_e.xml"

#                    [[[[[[Kindersly, SK]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/city/sk-21_e.xml"

                    [[[[[[Osoyoos, BC]]]]]]
 
                        feed_url = "https://weather.gc.ca/rss/city/bc-69_e.xml"

                    [[[[[[Penticton, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-84_e.xml"

                    [[[[[[Kelowna, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-48_e.xml"

                    [[[[[[Vernon, BC]]]]]]

                        inline_forecast = no
                        feed_url = "https://weather.gc.ca/rss/city/bc-27_e.xml"

                [[[[[Public Alerts]]]]] # List of areas to get public alerts for.

                    [[[[[[Okanagan Valley]]]]]]

                        feed_url ="https://weather.gc.ca/rss/battleboard/bc30_e.xml"

                    [[[[[[Nicola]]]]]]

                        feed_url = "https://weather.gc.ca/rss/battleboard/bc32_e.xml"

                    [[[[[[Similkameen]]]]]]

                        feed_url ="https://weather.gc.ca/rss/battleboard/bc31_e.xml"

                [[[[[Marine Areas]]]]]  # A list of marine areas to get weather and advisories for.

                # A feed may contain weather and advisories for more than one area so 
                # the area name is used to select the area of interest.  This means you 
                # MUST enter an area name exactly as it is found in the feed, which
                # should be the same as found on the Environment Canada web page.

#                    [[[[[[Queen Charlotte Strait]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/12400_e.xml"

#                    [[[[[[Juan de Fuca Strait - west entrance]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/07000_e.xml"

#                    [[[[[[Juan de Fuca Strait - central strait]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/07000_e.xml"

#                    [[[[[[Strait of Georgia - north of Nanaimo]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/14300_e.xml"

#                    [[[[[[Strait of Georgia - south of Nanaimo]]]]]]

#                        feed_url = "https://weather.gc.ca/rss/marine/14300_e.xml"

#            [[[[Drive BC]]]]

                # Drive BC provides information about road conditions, incidents and other events.

                # You can change where the Drive BC data is displayed by selecting a 
                # Belchertown skin include file:

                #belchertown_include_file = "index_hook_after_station_info.inc"
                #belchertown_include_file = "index_hook_after_forecast.inc"
                #belchertown_include_file = "index_hook_after_snapshot.inc"
#                belchertown_include_file = "index_hook_after_charts.inc"

                # include_file_sequence

#                include_file_sequence = only
                #include_file_sequence = first
                #include_file_sequence = subsequent
                #include_file_sequence = last

#                display_links_to_drive_bc = both # Display a link to Drive BC: top, bottom or both.

                # Map Center.

                # 'Position optionals' may be used here.

#                latitude = "49.1555"
#                longitude = "-119.5262" 
#                zoom = "12"

#                dbc_district_filter         = ""   # Select district: "district | district | district"

#                dbc_route_filter            = ""   # Select route: "rout | route | route"

                #dbc_type_filter             = ""   # Select event type.
                #dbc_type_filter             = "Incident"
                #dbc_type_filter             = "Current Planned"
                #dbc_type_filter             = "Future Planned"
                #dbc_type_filter             = "Road Condition"
#                dbc_type_filter             = "Incident | Current Planned | Road Condition"

#                dbc_severity_filter         = ""   # Select severity.

#                display_event_attributes        = "yes"    # Display event attributes for each event.
#                display_event_attribute_lists   = "yes"    # Display a list of event attributes.
                                                            # The purpose of these control is to display attributes 
                                                            # that can be used in the above filters.

#                [[[[[Areas]]]]]    # List of areas.
                                    # Areas are aggregated together into a single list without duplicates.
                                    # Selecting 'Entire Province' with another area works but is wasteful.

#                    [[[[[[Entire Province]]]]]]
#                    [[[[[[Thompson Okanagan]]]]]]
#                    [[[[[[Kootenays]]]]]]

            [[[[NOAA NWS - Forecasts]]]]

                # NOAA NWS provides feeds for forecasts for the USA.
                #
                # As above, NOAA NWS Forecast uses a unique grid system to retrieve a forecast.
                # NOAA NWS provides a service to translate latitude / longitude to the NOAA 
                # NWS grid system.
                #
                # You can manually enter latitude and longitude or you can allow Google Maps 
                # to look up latitude and longitude using the Google Maps Places service.
                #
                # Once latitude and longitude are known, WxFeeds using the NOAA NWS Points 
                # service to get the NOAA NWS Forecast grid values for the location of interest.
                #
                # <<Station Location>> may be used to get a forecast for your weather station 
                # location, if you have entered latitude and longitude for it.
                #
                # 'Position optionals' may entered for each Location.

                units = us    # Select units = 'us' (imperial) or 'si' (metric).
                              # May be set individually for each Location.

                [[[[[Locations]]]]] # List of Locations to get forecasts for.

                    # These controls set defaults for a list of Locations.
                    #
                    # Each control can be set individually for each Location.

                    get_forecast            = yes   # Get forecast by day - there are usally two entries for each day.
                    get_hourly_forecast     = no    # Get hourly forecast - the maximum number of hours is {need to research!}.
                    hourly_forecast_hours   = 24    # Number of hours to display for hourly forecast.

                    [[[[[[<<Station Location>>]]]]]]

                    [[[[[[Oroville, WA]]]]]]

                        get_forecast		= yes
                        get_hourly_forecast = yes
                        units 				= si

                    [[[[[[Seattle, WA]]]]]]

                        get_forecast 			= yes
                        get_hourly_forecast 	= no
                        hourly_forecast_hours 	= 36

#            [[[[NOAA NWS - Storm Prediction Center]]]]

                # NOAA NWS Storm Prediction Center issues weather alerts for land locations 
                # and marine areas.
                #
                # URLs for alert feeds can be found at https://alerts.weather.gov.
				# Alerts are offered by Zone and/or County.
				# Maps of NOAA zones cane be found at https://www.weather.gov/pimar/PubZone
				# but it looks like not all zones are on the maps.  SoI have not found 
				# a good, reliable way to identify the geographic area for a NOAA zone.  
				# The NOAA feed does have data fields for geographic infortation but 
				# I have not seen a feed with data so don't know if it can be used.
				# Perhaps I'll test for non-empty data and log something if detected.
            
                # NOAA NWS SPC also maintains list of all alerts.  Use this control to 
                # display a link to all current alerts.
				#
				# 'Position optionals' may be entered for each Alert.

#               display_links_to_all_current_alerts = "after"

#                [[[[[Alerts]]]]]    # A list of locations to get alerts for.

#                    [[[[[[Okanogan County]]]]]] # Location name.  This value MUST be unique!
                                                 # If not unique, weewx will not start.
                                                 # There are zones with the same name so I append 
                                                 # the zone ID to create a unique value.

#                        feed_url = "https://alerts.weather.gov/cap/wwaatmget.php?x=WAC047&y=0"

#            [[[[Washington State - Department of Transportation]]]]

                # The Washington State Department of Transportion provides several feeds for 
                # traveller information.
                #
                # Currently, only "border wait times" has been implemented.
                #
                # 'Position optionals' may be used for some services / subjects.

#                belchertown_include_file = "index_hook_after_station_info.inc"
                #belchertown_include_file = "index_hook_after_forecast.inc"
                #belchertown_include_file = "index_hook_after_snapshot.inc"
                #belchertown_include_file = "index_hook_after_charts.inc"

                #include_file_sequence = "only"
                #include_file_sequence = "first"
#                include_file_sequence = "subsequent"
                #include_file_sequence = "last"

#                ws_dot_access_code = "5809abc2-6afb-4ab7-bca1-7be1695b7086"
#                ws_dot_use_local_cache = "no"
#                ws_dot_cache_update_period_minutes = 1000000
#                ws_dot_force_cache_update = "yes"

#                [[[[[Traveler Information]]]]] # List of various traveller information subjects.

#                    [[[[[[Border Crossings]]]]]]

                      # 'Position optionals' may be used to set map zoom.

#            [[[[511 Alberta]]]]

                # Very similar to Drive BC.
                #
                # This section does NOT currently display any maps for any resource and 
                # needs to be enhanced same as Drive BC.

                #belchertown_include_file ="index_hook_after_station_info.inc"
                #belchertown_include_file ="index_hook_after_forecast.inc"
                #belchertown_include_file ="index_hook_after_snapshot.inc"
#                belchertown_include_file ="index_hook_after_charts.inc"

                #include_file_sequence = "only"
                #include_file_sequence = "first"
#                include_file_sequence = "subsequent"
                #include_file_sequence = "last"

#                [[[[[Resources]]]]]

#                    [[[[[[Road Conditions]]]]]]

#                        area_filter = "BANFF | JASPER"

#                    [[[[[[Alerts]]]]]]

#                    [[[[[[Events]]]]]]

#                        event_type_filter = "accidentsAndIncidents"
#                        roadway_name_filter = "HWY-16"

#                    [[[[[[Ferries]]]]]]

#                    [[[[[[Parks]]]]]]

                        #park_name_filter_condition = "equal"
#                        park_name_filter_condition = "contains"

#                        park_name_filter = "Sylvan | Big | Bow"

#                    [[[[[[Cameras]]]]]]

#                        display_disabled_cameras = "no"
#                        roadway_name_filter = "Hwy 1 | Hwy 2"

#                    [[[[[[Grouped Cameras]]]]]]

#                        display_disabled_cameras = "no"
#                        roadway_name_filter = "Hwy 1 | Hwy 2"

#            [[[[NOAA NWS - National Hurricane Center]]]]

                # Hurricane information from NOAA NWS - National Hurricane Center.
                #
                # Information about hurrincanes can be very large!
                #
                # 'Position optionals' are not used.

#                [[[[[Basins]]]]]

#                    [[[[[[Atlantic Basin Tropical Cyclones]]]]]]

#                        feed_url = "https://www.nhc.noaa.gov/index-at.xml"
