## page was renamed from ConfigExamples/php redirects
##master-page:CategoryTemplate
#format wiki
#language en
## This is a template for helping with new configuration examples. Remove this comment and add some descriptive text. A title is not necessary as the WikiPageName is already added here.
= Using PHP for Redirects =
<<Include(ConfigExamples, , from="^## warning begin", to="^## warning end")>>

<<TableOfContents>>

== Why PHP? ==
When looking at the documentation for squid I was relieved to find the page regarding [[/Features/Redirectors|redirecting]] urls. However, when I finally got there I was disappointed to see that it was in [[http://perl.org|perl]]. Not knocking perl, I know it is a more clean language than PHP, but I have little knowledge of the language, which limited my ability to edit and manipulate the redirect as I have done here.

== Getting Started ==
It should be noted that your redirect should be executable.

I will be placing the redirect in my squid configuration directory (/etc/squid), this is not a requirement, but its easier to find when you want to go back after a few months and make a few changes.

Don't forget to add the redirect program to your squid configuration.

{{{
url_rewrite_program /etc/squid/redirect.php
}}}
== PHP Redirect (Simple) ==
{{{
#!plain
#!/usr/bin/php
<?php

$temp = array();

while ( $input = fgets(STDIN) ) {
  // Split the output (space delimited) from squid into an array.
  $temp = split(' ', $input);

  // Set the URL from squid to a temporary holder.
  $output = $temp[0] . "\n";

  // Check the URL and rewrite it if it matches limewire.com
  if ( strpos($temp[0], "limewire.com") ) {
    $output = "302:http://www.google.com/\n";
  }
  echo $output;
}
}}}
As you can probably see this rewrites the URL so that it redirects any request that contains 'limewire.com' to the google homepage. I choose the 302 HTTP code so that the url in the users browser also reflects the redirect, otherwise it would display google's page and the url would read limewire.com.

I am fully aware that this is much more code than the perl example. Where this comes into play is later when we start looking at the requesting IP and the time, or day the request was made.

== PHP Redirect (With IP Checking) ==
{{{
#!plain
#!/usr/bin/php
<?php

$temp = array();

while ( $input = fgets(STDIN) ) {
  // Split the output (space delimited) from squid into an array.
  $temp = split(' ', $input);

  // Set the URL from squid to a temporary holder.
  $output = $temp[0] . "\n";

  // Clean the Requesting IP Address field up.
  $ip = rtrim($temp[1], "/-");

  // Check the requesting IP Address.
  if ( $ip == "192.168.0.101" ) {
    // Check the URL and rewrite it if it matches limewire.com
    if ( strpos($temp[0], "limewire.com") ) {
      $output = "302:http://www.google.com/\n";
    }
  }

  echo $output;
}
}}}
This redirect takes the requesting IP Address into consideration. It does not take much editing to change the way this works. I'll leave it to you to figure out how to use this to your advantage. (Hint: '==' equal, '!=' not equal)

== PHP Redirect (With day of week and time of day) ==
{{{
#!plain
#!/usr/bin/php
<?php

$temp = array();

while ( $input = fgets(STDIN) ) {
  // Split the output (space delimited) from squid into an array.
  $temp = split(' ', $input);

  // Set the URL from squid to a temporary holder.
  $output = $temp[0] . "\n";

  // Get the current day of the week and the current hour.
  $hour = date('G');
  $day = date('N');

  // If it is Monday - Friday and 8:00 - 17:00 (5:00 PM)
  if ( $hour >= 8 && $hour <= 16 && $day <= 5 ) {
    // Check the URL and rewrite it if it matches limewire.com
    if ( strpos($temp[0], "limewire.com") ) {
      $output = "302:http://www.google.com/\n";
    }
  }

  echo $output;
}
}}}
This will perform the redirect but only during a work week and during work hours.

== Putting it all together ==
{{{
#!plain
#!/usr/bin/php
<?php

$temp = array();

while ( $input = fgets(STDIN) ) {
  // Split the output (space delimited) from squid into an array.
  $temp = split(' ', $input);

  // Set the URL from squid to a temporary holder.
  $output = $temp[0] . "\n";

  // Clean the Requesting IP Address field up.
  $ip = rtrim($temp[1], "/-");

  // Get the current day of the week and the current hour.
  $hour = date('G');
  $day = date('N');

  if ($hour >= 8 && $hour <= 16 && $day <= 5) {
    if ( strpos($temp[0], "pogo.com") ) {
      $output = "302:http://www.google.com/\n";
    } elseif ( strpos($temp[0], "addictinggames.com") ) {
      $output = "302:http://www.google.com/\n";
    } elseif ( strpos($temp[0], "bigfishgames.com") ) {
      $output = "302:http://www.google.com/\n";
    } elseif ( strpos($temp[0], "games.yahoo.com") ) {
      $output = "302:http://www.yahoo.com/\n";
    } elseif ( strpos($temp[0], "facebook.com") ) {
      $output = "302:http://www.google.com/\n";
    } elseif ( strpos($temp[0], "myspace.com") ) {
      $output = "302:http://www.google.com/\n";
    }
  }
  if ( strpos($temp[0], "limewire.com") ) {
    $output = "302:http://www.google.com/\n";
  } elseif ( $ip == "192.168.0.101" && strpos($temp[0], "finance.yahoo.com") ) {
    $output = "302:http://finance.google.com/\n";
  }

  echo $output;
}
}}}
This is rather interesting, it redirects some flash gaming sites and two major social sites, but only during work hours. It also redirects limewire, and forces a machine to use google instead of yahoo for all their stock quotes!

----
 . CategoryConfigExample
