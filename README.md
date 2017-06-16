# nfsen-ng
[![GitHub license](https://img.shields.io/github/license/mbolli/nfsen-ng.svg?style=flat-square)](https://github.com/mbolli/nfsen-ng/blob/master/LICENSE)
[![GitHub issues](https://img.shields.io/github/issues/mbolli/nfsen-ng.svg?style=flat-square)](https://github.com/mbolli/nfsen-ng/issues)
[![Donate a beer](https://img.shields.io/badge/paypal-donate-yellow.svg?style=flat-square)](https://paypal.me/bolli)

nfsen-ng is an in-place replacement for the ageing nfsen.

<div><img src="https://user-images.githubusercontent.com/722725/26952171-ee1ac112-4cac-11e7-842d-ccefbda9b287.png" style="width: 100%" /></div>

**Used components**

 * Front end: [jQuery](https://jquery.com), [dygraphs](http://dygraphs.com), [FooTable](http://fooplugins.github.io/FooTable/), [ion.rangeSlider](http://ionden.com/a/plugins/ion.rangeSlider/en.html)
 * Back end:  [RRDtool](http://oss.oetiker.ch/rrdtool/), [nfdump tools](https://github.com/phaag/nfdump), 

## TOC

   * [nfsen-ng](#nfsen-ng)
      * [Installation](#installation)
      * [Configuration](#configuration)
      * [CLI](#cli)
      * [API](#api)
         * [/api/config](#apiconfig)
         * [/api/graph](#apigraph)
         * [/api/flows](#apiflows)
         * [/api/stats](#apistats)

## Installation

Ubuntu 16.04 LTS:
 
 ```sh
 apt-get install apache2 php7.0 php7.0-dev libapache2-mod-php7.0 pkg-config nfdump rrdtool librrd-dev
 a2enmod rewrite deflate headers expires
 pecl install rrd # install rrd library for php
 cd /etc/php/7.0/mods-available && vim rrd.ini  # add extension=rrd.so
 phpenmod rrd
 vim /etc/apache2/apache2.conf # set AllowOverride All for /var/www
 service apache2 restart
 cd /var/www/html # or wherever
 git clone https://github.com/mbolli/nfsen-ng
 chown -R www-data:www-data .
 ```
 
 Fedora/CentOS: (TBD)

## Configuration
The default settings file is `backend/settings/settings.php.dist`. Copy it to `backend/settings/settings.php` and start modifying it:

 * **general**
    * **ports:** (_array(80, 23, 22, ...)_) The ports to examine. _Note:_ If you use RRD as datasource and want to import existing data, you might keep the number of ports to a minimum, or the import time will be measured in moon cycles...
    * **sources:** (_array('source1', ...)_) The sources to scan. 
    * **db:** (_RRD_) The name of the datasource class (case-sensitive).
 * **nfdump**
    * **binary:** (_/usr/bin/nfdump_) The location of your nfdump executable
    * **profiles-data:** (_/var/nfdump/profiles_data_) The location of your nfcapd files
    * **profile:** (_live_) The profile folder to use
    * **max-processes:** (_1_) The maximum number of concurrently running nfdump processes. _Note:_ Statistics and aggregations can use lots of system resources, even to aggregate one week of data might take more than 15 minutes. Put this value to > 1 if you want nfsen-ng to be usable while running another query.
 * **db** If the used data source needs additional configuration, you can specify it here, e.g. host and port.
 * **log**
    * **priority:** (_LOG_INFO_) see other possible values at [http://php.net/manual/en/function.syslog.php]
    
## CLI

The command line interface is used to initially scan existing nfcapd.* files, or to administer the daemon.

Usage: 
  
  `./cli.php [ options ] import`
or  
  `./cli.php start|stop|status`


 * **Options:**
     * **-v**  Show verbose output
     * **-p**  Import ports data as well _Note:_ Using RRD this will take quite a bit longer, depending on the number of your defined ports.
     * **-ps**  Import ports per source as well _Note:_ Using RRD this will take quite a bit longer, depending on the number of your defined ports.
     * **-s** Skip importing sources data
     * **-f**  Force overwriting database and start fresh

 * **Commands:**
     * **import** Import existing nfdump data to nfsen-ng. _Note:_ If you have existing nfcapd files, better do this overnight.
     * **start** Start the daemon for continuous reading of new data
     * **stop** Stop the daemon
     * **status** Get the daemon's status
        
 * **Examples:**
     * `./cli.php -f import`
        Imports fresh data for sources

     * `./cli.php -s -p import`
        Imports data for ports only

     * `./cli.php start`
        Starts the daemon
        
        
## API
The API is used by the frontend to retrieve data. 

### /api/config
 * **URL**
    `/api/config`
 
 * **Method:**
    `GET`
   
 *  **URL Params**
     none
 
 * **Success Response:**
 
    * **Code:** 200 
     **Content:** 
        ```json
        {
          "sources": [ "gate", "swi6" ],
          "ports": [ 80, 22, 23 ],
          "stored_output_formats": [], 
          "stored_filters": [],
          "daemon_running": true
        }
        ```
  
 * **Error Response:**
 
   * **Code:** 400 BAD REQUEST 
     **Content:** 
        ```json 
        {"code": 400, "error": "400 - Bad Request. Probably wrong or not enough arguments."}
        ```
    OR
 
    * **Code:** 404 NOT FOUND 
     **Content:** 
        ```json
        {"code": 404, "error": "400 - Not found. "}
        ```
 
 * **Sample Call:**
    ```sh
    curl localhost/nfsen-ng/api/config
    ```
 
### /api/graph
* **URL**
  `/api/graph?datestart=1490484000&dateend=1490652000&type=flows&sources[0]=gate&protocols[0]=tcp&protocols[1]=icmp`

* **Method:**
  
  `GET`
  
*  **URL Params**
    * `datestart=[integer]` Unix timestamp
    * `dateend=[integer]` Unix timestamp
    * `type=[string]` Type of data to show: flows/packets/bytes
    * `sources=[array]` 
    * `protocols=[array]`
    * `ports=[array]`
    
    There can't be multiple sources and multiple protocols both. Either one source and multiple protocols, or one protocol and multiple sources.

* **Success Response:**
  
  * **Code:** 200
    **Content:** 
    ```json 
    {"data": {
      "1490562300":[2.1666666667,94.396666667],
      "1490562600":[1.0466666667,72.976666667],...
    },"start":1490562300,"end":1490590800,"step":300,"legend":["swi6_flows_tcp","gate_flows_tcp"]}
    ```
 
* **Error Response:**  
  * **Code:** 400 BAD REQUEST <br />
      **Content:** 
        ```json
        {"code": 400, "error": "400 - Bad Request. Probably wrong or not enough arguments."}
        ```
  
   OR
  
  * **Code:** 404 NOT FOUND <br />
      **Content:** 
        ```json 
        {"code": 404, "error": "400 - Not found. "}
        ```

* **Sample Call:**

  ```sh
  curl -g "http://localhost/nfsen-ng/api/graph?datestart=1490484000&dateend=1490652000&type=flows&sources[0]=gate&protocols[0]=tcp&protocols[1]=icmp"
  ```

### /api/flows
* **URL**
  `/api/flows?datestart=1482828600&dateend=1490604300&sources[0]=gate&sources[1]=swi6&filter=&limit=100&aggregate=srcip&sort=&output[format]=auto`

* **Method:**
  
  `GET`
  
*  **URL Params**
    * `datestart=[integer]` Unix timestamp
    * `dateend=[integer]` Unix timestamp
    * `sources=[array]` 
    * `filter=[string]` pcap-syntaxed filter
    * `limit=[int]` max. returned rows
    * `aggregate=[string]` can be `bidirectional` or a valid nfdump aggregation string (e.g. `srcip4/24, dstport`), but not both at the same time
    * `sort=[string]` (will probably cease to exist, as ordering is done directly in aggregation) e.g. `tstart`
    * `output=[array]` can contain `[format] = auto|line|long|extended` and `[IPv6]` 

* **Success Response:**
  
  * **Code:** 200
    **Content:** 
    ```json 
    [["ts","td","sa","da","sp","dp","pr","ipkt","ibyt","opkt","obyt"],
    ["2017-03-27 10:40:46","0.000","85.105.45.96","0.0.0.0","0","0","","1","46","0","0"],
    ...
    ```
 
* **Error Response:**
  
  * **Code:** 400 BAD REQUEST <br />
      **Content:** 
        ```json
        {"code": 400, "error": "400 - Bad Request. Probably wrong or not enough arguments."}
        ```
  
   OR
  
  * **Code:** 404 NOT FOUND <br />
      **Content:** 
        ```json 
        {"code": 404, "error": "400 - Not found. "}
        ```

* **Sample Call:**

  ```sh
  curl -g "http://localhost/nfsen-ng/api/flows?datestart=1482828600&dateend=1490604300&sources[0]=gate&sources[1]=swi6&filter=&limit=100&aggregate[]=srcip&sort=&output[format]=auto"
  ```

### /api/stats
* **URL**
  `/api/stats?datestart=1482828600&dateend=1490604300&sources[0]=gate&sources[1]=swi6&for=dstip&filter=&top=10&limit=100&aggregate[]=srcip&sort=&output[format]=auto`

* **Method:**
  
  `GET`
  
*  **URL Params**
    * `datestart=[integer]` Unix timestamp
    * `dateend=[integer]` Unix timestamp
    * `sources=[array]` 
    * `filter=[string]` pcap-syntaxed filter
    * `top=[int]` return top N rows
    * `for=[string]` field to get the statistics for. with optional ordering field as suffix, e.g. `ip/flows`
    * `limit=[string]` limit output to records above or below of `limit` e.g. `500K`
    * `output=[array]` can contain `[IPv6]` 

* **Success Response:**
  
  * **Code:** 200
    **Content:** 
    ```json 
    [
        ["Packet limit: > 100 packets"],
        ["ts","te","td","pr","val","fl","flP","ipkt","ipktP","ibyt","ibytP","ipps","ipbs","ibpp"],
        ["2017-03-27 10:38:20","2017-03-27 10:47:58","577.973","any","193.5.80.180","673","2.7","676","2.5","56581","2.7","1","783","83"],
        ...
    ]
    ```
 
* **Error Response:**
  
  * **Code:** 400 BAD REQUEST <br />
      **Content:** 
        ```json
        {"code": 400, "error": "400 - Bad Request. Probably wrong or not enough arguments."}
        ```
  
   OR
  
  * **Code:** 404 NOT FOUND <br />
      **Content:** 
        ```json 
        {"code": 404, "error": "400 - Not found. "}
        ```

* **Sample Call:**

  ```sh
  curl -g "http://localhost/nfsen-ng/api/stats?datestart=1482828600&dateend=1490604300&sources[0]=gate&sources[1]=swi6&for=dstip&filter=&top=10&limit=100&aggregate[]=srcip&sort=&output[format]=auto"
  ```

More endpoints to come:
* `/api/graph_stats`
