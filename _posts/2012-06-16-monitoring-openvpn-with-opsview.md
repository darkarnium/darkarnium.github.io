---
layout: post
title: Monitoring OpenVPN with OpsView
date: 2012-06-16 16:41:50.000000000 -07:00
author: Peter Adkins
image:
---
While working on monitoring critical services at $company, I have recently had a requirement to monitor an OpenVPN instance to ensure that it is available when required. In order to do so, I have compiled a "quick and dirty" plugin for OpsView / Nagios. This script was compiled in a number of hours using Wireshark to capture the initial client -&gt; server handshake which was then binary encoded and replayed via Perl.

Again, this is crude service check, but it may assist someone in the future.

{% highlight perl %}
#!/usr/bin/perl
#
# Filaname: check_openvpn
# Created:  2012-06-15
# Website:  http://blog.kernelpicnic.net
#
# Description:
# This script is for verifying the status of an OpenVPN daemon. It has been
# written to integrate directly with Nagios / Opsview.
#
# Usage:
#    check_openvpn [OPTIONS]...
#
#      -H, --hostname      Host to check
#      -p, --port          Port number to check
#      -h, --help          Display help.
#
#############################################################################
# Custom library path for Nagis modules.
use lib qw ( /usr/local/nagios/perl/lib );

# Enforce sanity.
use strict;
use warnings;

# Required modules.
use Getopt::Long qw(:config no_ignore_case);
use Nagios::Plugin;
use IO::Socket;

# Define defaults.
my $help    = 0;
my $timeout = 5;

# Ensure required variables are set.
my($hostname, $port);
my $options = GetOptions(
                         "hostname|H=s" => $hostname,
                         "timeout|t=s"  => $timeout,
                         "port|p=s"     => $port,
                         "help|h"       => $help,
);

# Check if help has been requested.
if($help || !$hostname || !$port) {
    printf("n");
    printf("Usage: check_openvpn [OPTIONS]...nn");
    printf("  -H, --hostname      Host to checkn");
    printf("  -p, --port          Port number to checkn");
    printf("  -h, --help          This help pagen");
    printf("  -t, --timeout       Socket timeoutn");
    printf("n");
    exit(-1);
}

# Setup a new Nagios::Plugin object.
my $nagios = Nagios::Plugin->new();

# Define the check string to send to the OpenVPN server - as binary due
# to non-printable characters.
my $check_string = "001110000011001010010010011011101000000100010001110"
                  ."100110110101010110011000000000000000000000000000000"
                  ."0000000000";

# Attempt to setup a socket to the specified host.
my $host_sock = IO::Socket::INET->new(
                                      Proto    => 'udp',
                                      PeerAddr => $hostname,
                                      PeerPort => $port,
);

# Ensure we have a socket.
if(!$host_sock) {
    $nagios->nagios_exit(UNKNOWN, "Unable to bind socket");
}

# Fire off the check request.
$host_sock->send(pack("B*", $check_string));

# Wait for $timeout for response for a response, otherwise, fail.
my $response;
eval {

    # Define how to handle ALARM.
    local $SIG{ALRM} = sub {
        $nagios->nagios_exit(CRITICAL, "No response received");
    };

    # Set the alarm for the given timeout value.
    alarm($timeout);

    # Check for response.
    $host_sock->recv($response, 1)
      or $nagios->nagios_exit(CRITICAL, "No response received");

    # Alright, response received, cancel alarm.
    alarm(0);
    1;

};

# Reply received, return okay.
$nagios->nagios_exit(OK, "Response received from host");
{% endhighlight %}
