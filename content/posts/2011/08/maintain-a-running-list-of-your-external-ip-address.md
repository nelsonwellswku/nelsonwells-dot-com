+++
title = 'Maintain a running list of your external IP address'
date = 2011-08-07T15:00:00-05:00
draft = false
tags = ['programming', 'perl', 'cron', 'linux']
+++
At home, I have a consumer level cable internet service.  I actually do not know if I have a dynamic IP address or a static IP address because I haven't really cared enough to ask my ISP.  I would assume that I have a dynamic IP address since most ISPs charge a premium for a static IP address, but since I don't know for sure I thought I would find out for myself.

Using http://whatismyip.com 's "automation" page, Perl, and cron, I am going to maintain a list of my external IP addresses.  Not only will this answer the original question, but I can also gather some additional information, such as how often my IP address changes and whether or not I can force my IP address to change by disconnecting my modem :).

<!--more-->

## The logical steps

The method I am going to use is relatively simple.  First, I need a script that will fetch a page with my current external IP address.  I'm going to use Perl, but any scripting language will do.  Once the script has fetched my current IP address, I will append the IP address and the current date, separated by a comma, to a log file.  I want the two fields to be comma separated in case I ever want to run a script over the log it will be easy to grab the information I want.

Once the script is finished, I need a way to automate it.  I would like to do this once a day... you may want to do this less often or more often depending on your goal.  Luckily, this is simple to do with cron, the Linux utility to run scheduled tasks.

## The code

First, we need to create the script.  I am going to call my script ip_logger.pl.  We need to declare that the file is a Perl script, give the usual safety directives, and load relevant libraries.

    #!/usr/bin/perl
    use strict;
    use warnings;
    use File::Slurp;
    use LWP::Simple;
    use Time::localtime;

I've loaded `File::Slurp` to ease reading and writing the the log file.  I've loaded `LWP::Simple` (libwww-perl) to fetch the page with our external IP.  I use Simple since we just need to make a GET request, we don't need any additional features.  I've loaded `Time::localtime` to make generating a current date string simple and easy to read.

Next, we need to decide where we're going to put our log file, and then actually make the request to the website to retrieve our external IP address.

    # cron runs the script from your home directory, 
    # so give an absolute path
    my $ip_addresses_filename = '/home/nelson/projects/perl/ip_logger/ip_addresses.txt';

    # get current external ip address
    my $content = get('http://automation.whatismyip.com/n09230945.asp');
    $content = 'Could not pull ip address!' if !$content;

Let's notice that we create a scalar called $content and populate it with the contents at the whatismyip.com 's automation page.  The page has no HTML, it only contains the IP address, so we don't have to parse the IP address or do anything else.  We'll just assign the returned string to the scalar.  Also notice that if the page fetch fails, we'll put a string indicative of such in the `$content` scalar.

Next, we need to capture the current date.  I'm going for the month-day-year format.

    # get the current date to also write to our file
    my $month = localtime->mon + 1;
    my $day = localtime->mday;
    my $year = localtime->year + 1900;
    my $current_date = "$month-$day-$year";

The last thing the script needs to do is actually write to the log file.  File::Slurp's `write_file` function is smart enough to create the file if it doesn't exist so we don't need to guard against that.

    # write / append it to the file
    write_file($ip_addresses_filename, {append => 1}, $content . ',' . $current_date . "\n");

## The cron
The script by itself will not do a whole lot of good.  Unless you want to manually run the script once a day, we need to schedule it to run automatically.  In shell, we will do crontab -e to open our crontab for editing.  Then, we'll insert this line.

    30 6 * * * /home/nelson/projects/perl/ip_logger/ip_logger.pl
I want to run the script at 6:30 AM every day.  As I said earlier, you may want to adjust the interval to something more appropriate for your situation.

## The result

After a few days, I'll have a log file that looks similar to this.

    104.94.155.241,8-8-2011
    104.94.155.241,8-9-2011
    104.94.155.241,8-10-2011
    104.94.155.241,8-11-2011
    104.94.155.241,8-12-2011
    104.94.155.241,8-13-2011

Happy automating!
