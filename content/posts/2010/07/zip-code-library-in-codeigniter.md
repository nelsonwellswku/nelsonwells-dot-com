+++
title = 'Zip code library in CodeIgniter'
date = 2010-07-07T15:00:00-05:00
draft = false
tags = ['programming', 'php', 'codeigniter']
+++
Recently, a client wanted a member search for his website that included a search by zip code.  The 'easy' solution would be to implement the search as a LIKE statement in SQL, but the solution is inaccurate, and I like the 'good' solutions over the 'easy' ones.  I looked for a CodeIgniter library that would give me the functionality for which I wanted to implement, but no such thing existed.  Then, I came across Micah Carrick's native PHP library which gave the exact functionality I was looking for.  I did end up using his library, but not without modification: I've ported the native library to a CodeIgniter library.  This is where you can get it.

<!--more-->

<h3>The Code</h3>

First, you'll need to create your database.  You can use this MySQL script to create the table.

    CREATE TABLE `zip_code` (
    `id` int(11) unsigned NOT NULL auto_increment,
    `zip_code` varchar(5) collate utf8_bin NOT NULL,
    `city` varchar(50) collate utf8_bin default NULL,
    `county` varchar(50) collate utf8_bin default NULL,
    `state_name` varchar(50) collate utf8_bin default NULL,
    `state_prefix` varchar(2) collate utf8_bin default NULL,
    `area_code` varchar(3) collate utf8_bin default NULL,
    `time_zone` varchar(50) collate utf8_bin default NULL,
    `lat` float NOT NULL,
    `lon` float NOT NULL,
    PRIMARY KEY  (`id`),
    KEY `zip_code` (`zip_code`)

You will need to populate the table, of course, and you will do so with the MySQL scripts included in the project.

Now, for the meat and potatoes.  Here is the library itself.  It is GPL Version 3 licensed so keep that in mind if you decide to use it.

    <?php

    if (!defined('BASEPATH'))
        exit('No direct script access allowed');

    //constant for convering to kilometers from miles
    define('M2KM_FACTOR', 1.609344);

    // constants for passing $sort to get_zips_in_range()
    define('SORT_BY_DISTANCE_ASC', 1);
    define('SORT_BY_DISTANCE_DESC', 2);
    define('SORT_BY_ZIP_ASC', 3);
    define('SORT_BY_ZIP_DESC', 4);

    class Geozip
    {
        var $units;
        var $decimals;
        var $last_error;
        var $CI;

        function __construct()
        {
            $this->CI = get_instance();

            $this->units = "miles";
            $this->decimals = 2;
        }

        function get_zip_details($zip)
        {
            $this->CI->db->select("lat AS lattitude, lon AS longitude, city, county, state_prefix");
            $this->CI->db->select("state_name, area_code, time_zone");
            $this->CI->db->where("zip_code", $zip);
            $results = $this->CI->db->get("zip_code");

            if ($results->num_rows() < 1) {
                $this->set_last_error("Zip not found in database");
                return false;
            } else {
                return $results->row();
            }
        }

        function get_zips_in_range($zip, $range, $sort = 1, $include_base = true)
        {
            //get base zip details
            $details = $this->get_zip_point($zip);

            if (!$details) {
                return false;
            }

            //find max - min lat / long for radius and zero point and query
            //only zips in that range.
            $lat_range = $range / 69.172;
            $lon_range = abs($range / (cos($details->lat) * 69.172));
            $min_lat = number_format($details->lat - $lat_range, "4", ".", "");
            $max_lat = number_format($details->lat + $lat_range, "4", ".", "");
            $min_lon = number_format($details->lon - $lon_range, "4", ".", "");
            $max_lon = number_format($details->lon + $lon_range, "4", ".", "");

            //build the sql query
            $this->CI->db->select("zip_code, lat, lon");

            if (!$include_base) {
                $this->CI->db->where("zip_code <>", $zip);
            }

            $this->CI->db->where("lat BETWEEN '$min_lat' AND '$max_lat'");
            $this->CI->db->where("lon BETWEEN '$min_lon' AND '$max_lon'");

            $result = $this->CI->db->get("zip_code");

            if ($result->num_rows() < 1) {
                $this->set_last_error("SQL error in get_zips_in_range");
                return false;
            } else {
                //loop through all 40 some thousand zip codes and determine whether
                //or not it's within the specified range.

                foreach ($result->result() as $row) {
                    $distance = $this->calculate_mileage($details->lat, $row->lat, $details->lon, $row->lon);

                    if ($this->units == "kilos") {
                        $distance *= M2KM_FACTOR;
                    }

                    if ($distance <= $range) {
                        $zips[$row->zip_code] = $distance;
                    }
                }
            }

            //sort the zips as selected
            switch ($sort) {
                case SORT_BY_DISTANCE_ASC:
                    asort($zips);
                    break;

                case SORT_BY_DISTANCE_DESC:
                    arsort($zips);
                    break;

                case SORT_BY_ZIP_ASC:
                    ksort($zips);
                    break;

                case SORT_BY_ZIP_DESC:
                    krsort($zips);
                    break;
            }

            return $zips;

        }

        /*
        * Get the distance between 2 zip codes.
        */
        function get_distance($zip1, $zip2)
        {
            //return 0 miles / kilos if its the same zip
            if ($zip1 == $zip2) {
                return 0;
            }

            //get the details from the database and exit if there is an error
            $details1 = $this->get_zip_point($zip1);
            $details2 = $this->get_zip_point($zip2);

            if ($details1 === false || $details2 === false) {
                return false;
            }

            //calculate the distance between the 2 zip codes based on
            //the latitude and longitude pulled from the database
            $miles = $this->calculate_mileage(
                $details1->lat,
                $details2->lat,
                $details1->lon,
                $details2->lon
            );

            if ($this->units == "kilos") {
                return round($miles * M2KM_FACTOR, $this->decimals);
            } else {
                return round($miles, $this->decimals);
            }
        }

        /*
        * Set the units to describe distance
        * Accepts "miles" or "kilos"
        */
        function set_units($units = "miles")
        {
            if ($units != "kilos" || $units != "miles") {
                $this->units = "miles";
            } else {
                $this->units = $units;
            }
        }

        function get_last_error()
        {
            return $this->last_error();
        }

        /*
        * Pull latitude and longitude from the database
        */
        private function get_zip_point($zip)
        {
            $this->CI->db->select("lat, lon")->where("zip_code", $zip);
            $result = $this->CI->db->get("zip_code");

            if ($result->num_rows() < 1) {
                $this->set_last_error("Zip code not found in db: $zip");
                return false;
            } else {
                return $result->row();
            }
        }

        private function calculate_mileage($lat1, $lat2, $lon1, $lon2)
        {
            //convert lattitude/longitude (degrees) to radians for calculations
            $lat1 = deg2rad($lat1);
            $lon1 = deg2rad($lon1);
            $lat2 = deg2rad($lat2);
            $lon2 = deg2rad($lon2);

            //find the deltas
            $delta_lat = $lat2 - $lat1;
            $delta_lon = $lon2 - $lon1;

            //find the Great Circle distance
            $temp = pow(sin($delta_lat / 2.0), 2) + cos($lat1) * cos($lat2) * pow(sin($delta_lon / 2.0), 2);
            $distance = 3956 * 2 * atan2(sqrt($temp), sqrt(1 - $temp));

            return $distance;
        }

        private function set_last_error($error)
        {
            $this->last_error = $error;
        }
    }
    ?>

## How to use

The library supports 3 main features right now: get zip details, get distance between 2 zip codes, and get a list of zip codes in a given radius of another zip code.  The library supports miles and kilometers as distance units.  I'll demonstrate how to use each of these functions, and how to change the units.

Copy the code above into a file named geozip.php and place it in the application/libraries folder in your CodeIgniter installation.  Then, in whatever controller you want to use it in, load it like you would any other library.

    $this->load->library("geozip");

Getting and displaying the zip's details are simple.  In the controller, you would do

    $zip_details = $this->geozip->get_zip_details($this->input->post("zip"));
    $data['zip_details'] = $zip_details;

and then in the view, you could access the following variables:

    echo $zip_details->lattitude;
    echo $zip_details->longitude;
    echo $zip_details->city;
    echo $zip_details->county;
    echo $zip_details->state_prefix;
    echo $zip_details->state_name;
    echo $zip_details->area_code;
    echo $zip_details->time_zone;

Getting the distance between 2 zips is equally as easy.

    $distance = $this->geozip->get_distance("90210", "60601");

Doing a radial search is only slightly more complicated, but only because you have some options when you call the function.  The function accepts 4 parameters: the zip code, the range to cover, sort order, and whether or not to include the original zip in the result set.  The default sort order is sorted by distance ascending, and the default 'include base' is true.  You can change the default sort order using the following defines, and you can change the 'include base' by setting it to boolean false.

    // constants for passing $sort to get_zips_in_range()
    define('SORT_BY_DISTANCE_ASC', 1);
    define('SORT_BY_DISTANCE_DESC', 2);
    define('SORT_BY_ZIP_ASC', 3);
    define('SORT_BY_ZIP_DESC', 4);

Here is how you could find all the zips in a 20 mile radius of Beverly Hills sorted by zip ascending, but not including Beverly Hills.

    $zips = $this->geozip->get_zips_in_range("90210", 20, SORT_BY_ZIP_ASC, false);
    $data['zips'] = $zips;

`$zips` will be an associative array with the zip code as the key and the distance from the origin zip as the value.  You can process this in your controller or view as you see fit.

    foreach($zips as $zip => $distance)
    {
        echo "$zip is $distance miles away from the origin.";
        //or other processing you wish to do with these zips.
    }

I did say that the library supports miles and kilometers.  To change between them, simply do

    $this->geozip->set_units("kilos"); //only accepts "miles" or "kilos"

And any function that returns a distance will return whichever unit you've specified.

## Things to remember

The biggest problem with this approach is the database.  As stated in the original blog by Micah, the database was derived from a variety of sources and may be out of date.  Depending on how critical this feature is to your application, the current database may or may not work.  If you want an up to date database, you'll probably have to purchase and import it yourself.

Happy coding!
