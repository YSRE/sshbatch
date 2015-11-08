# NAME

SSH::Batch - Cluster operations based on parallel SSH, set and interval arithmetic

Table of Contents
=================

* [NAME](#name)
* [VERSION](#version)
* [SYNOPSIS](#synopsis)
* [DESCRIPTION](#description)
* [TIPS](#tips)
* [PREREQUISITES](#prerequisites)
* [INSTALLATION](#installation)
* [CODE REPOSITORY](#code-repository)
* [TODO](#todo)
* [AUTHORS](#authors)
* [COPYRIGHT & LICENSE](#copyright--license)
* [SEE ALSO](#see-also)

# VERSION

This document describes SSH::Batch 0.029 released on 29 April 2012.

# SYNOPSIS

The following scripts are provided:

- fornodes

    Expand patterns to machine host list.

        $ cat > ~/.fornodesrc
        ps=blah.ps.com bloo.ps.com boo[2-25,32,41-70].ps.com
        as=ws[1101-1105].as.com
        # use set operations to define new sets:
        foo={ps} + {ps} * {as} - {ps} / {as}
         bar = foo.com bar.org \
            bah.cn \
            baz.com
        ^D

        $ fornodes 'api[02-10].foo.bar.com' 'boo*.ps.com'
        $ fornodes 'tq[ab-ac].[1101-1105].foo.com'
        $ fornodes '{ps} + {as} - ws1104.as.com'  # set union and subtraction
        $ fornodes '{ps} * {as}'  # set intersect

- atnodes

    Run command on clusters. (atnodes calls fornodes internally.)

        # run a command on the specified servers:
        $ atnodes $'ps -fe|grep httpd' 'ws[1101-1105].as.com'

        # multiple-arg command requires "--":
        $ atnodes ls /opt/ -- '{ps} + {as}' 'localhost'

        # or use single arg command:
        $ atnodes 'ls /opt/' '{ps} + {as}' 'localhost' # ditto

        # specify a different user name and SSH server port:
        $ atnodes hostname '{ps}' -u agentz -p 12345

        # use -w to prompt for password if w/o SSH key (no echo back)
        $ atnodes hostname '{ps}' -u agentz -w

        # or prompt for password if both login and sudo are required...
        $ atnodes 'sudo apachectl restart' '{ps}' -w

        # or prompt for password for sudo only...
        $ atnodes 'sudo apachectl restart' '{ps}' -W

        # run sudo command if tty required...
        $ atnodes -tty 'sudo apachectl restart' '{ps}'

        # or specify a timeout:
        $ atnodes 'ping foo.com' '{ps}' -t 3

- tonodes

    Upload local files/directories to remote clusters

        $ tonodes /tmp/*.inst -- '{as}:/tmp/'
        $ tonodes foo.txt 'ws1105*' :/tmp/bar.txt

        # use rsync instead of scp:
        $ tonodes foo.txt 'ws1105*' :/tmp/bar.txt -rsync

        $ tonodes -r /opt /bin/* -- 'ws[1101-1102].foo.com' 'bar.com' :/foo/bar/

- key2nodes

    Push the SSH public key (or generate one if not any) to the remote clusters.

        $ key2nodes 'ws[1101-1105].as.com'

# DESCRIPTION

System administration (sysadmin) is also part of my `$work`. Playing with a (big) bunch of  machines without a handy tool is painful. So I refactored some of our old scripts and hence this module.

This is a high-level abstraction over the powerful [Net::OpenSSH](https://metacpan.org/pod/Net::OpenSSH) module. A bunch of handy scripts are provided to simplify big cluster operations: [fornodes](https://metacpan.org/pod/fornodes), [atnodes](https://metacpan.org/pod/atnodes), [tonodes](https://metacpan.org/pod/tonodes), and [key2nodes](https://metacpan.org/pod/key2nodes).

`SSH::Batch` allows you to name your clusters using variables and interval/set syntax in your `~/.fornodesrc` config file (or a different file name specified by the `SSH_BATCH_RC` environment). For instance:

    $ cat ~/.fornodesrc
    A=foo[01-03].com bar.org
    B=bar.org baz[a-b,d,e-g].cn foo02.com
    C={A} * {B}
    D={A} - {B}

where cluster `C` is the intersection set of cluster `A` and `B` while `D` is the sef of machines that are in `A` but not in `B`.

And then you can query machine host list by using `SSH::Batch`'s [fornodes](https://metacpan.org/pod/fornodes) script:

    $ fornodes '{C}'
    bar.org foo02.com

    $ fornodes '{D}'
    foo01.com foo03.com

    $ fornodes blah.com '{C} + {D}'
    bar.org blah.com foo01.com foo02.com foo03.com

It's always best practice to **put spaces around set operators** like `+`, `-`, `*`, and `/`, so as to allow these characters (notably the dash `-`) in your host names, as in:

    $ fornodes 'foo-bar-[a-d].com - foo-bar-c.com'
    foo-bar-a.com foo-bar-b.com foo-bar-d.com

for the ranges like `[a-z]`, there's also an alternative syntax:

    [a..z]

To exclude some discrete values from certain range, you need set subtration:

    foo[1-100].com - foo[32,56].com

or equivalently

    foo[1-31,33-55,57-100].com

[fornodes](https://metacpan.org/pod/fornodes) could be very handy in shell programming. For example, to test the 80 port HTTP service of a cluster `A`, simply put

    $ for node in `fornodes '{A}'`; \
        do curl "http://$node:80/blah'; \
      done

Also, other scripts in this module, like [atnodes](https://metacpan.org/pod/atnodes), [tonodes](https://metacpan.org/pod/tonodes), and [key2nodes](https://metacpan.org/pod/key2nodes) also call fornodes internally so that you can use the cluster spec syntax in those scripts' command line as well.

[atnodes](https://metacpan.org/pod/atnodes) meets the common requirement of running a command on a remote cluster. For example:

    # at the concurrency level of 6:
    atnodes 'ls -lh' '{A} + {B}' my.more.com -c 6

Or upload a local file to the remote cluster:

    tonodes ~/my.tar.gz '{A} / {B}' :/tmp/

or multiple files as well as some directories:

    tonodes -r ~/mydir ~/mydir2/*.so -- foo.com bar.cn :~/

It's also possible to use wildcards in the cluster spec expression, as in

    atnodes 'ls ~' 'api??.*.com'

where [atnodes](https://metacpan.org/pod/atnodes) will match the pattern `api??.*.com` against the "universal set" consisting of those hosts appeared in `~/fornodesrc` and those host names apeared before this pattern on the command line (if any). Note that only `?` (match any character) and `*` (match 0 or more characters) are supported here.

There's also a [key2nodes](https://metacpan.org/pod/key2nodes) script to push SSH public keys to remote machines ;)

[Back to TOC](#table-of-contents)

# TIPS

There's some extra tips found in our own's everyday use:

- Running sudo commands

    Often, we want to run commands requiring root access, such as when installing
    software packages on remote machines. So you'll have to tell [atnodes](https://metacpan.org/pod/atnodes) to
    prompt for your password:

        $ atnodes 'sudo yum install blah' '{my_cluster}' -w

    Then you'll be prompted by the `Password:` prompt after which you enter your
    remote password (with echo back turned off).

    Because the remote `sshd` might be smart enough to "remember" the sudo password
    for a (small) amount of time, immediate subsequent "sudo" might omit the `-w` option, as in

        $ atnodes 'sudo mv ~/foo /usr/local/bin/' {my_cluster}

    But remember, you can use _sudo without passwords_ just for a _small_ amount of
    time ;)

    If you see the following error message while doing sudo with [atnodes](https://metacpan.org/pod/atnodes)

        sudo: sorry, you must have a tty to run sudo

    then you should add option -tty, or you can probably comment out the "Defaults requiretty" line in your server's `/etc/sudoers` file (best just to do this for your own account).

- Passing custom options to the underlying `ssh`

    By default, `atnodes` relies on [Net::OpenSSH](https://metacpan.org/pod/Net::OpenSSH) to locate the OpenSSH client executable "ssh". But you can define the `SSH_BATCH_SSH_CMD` environment to specify the command explicitly. You can use the `-ssh` option to override it further. (The [key2nodes](https://metacpan.org/pod/key2nodes) script also supports the `SSH_BATCH_SSH_CMD` environment.)

    Note that to specify your own "ssh" is also a way to pass more options to the underlying OpenSSH client executable when using `atnodes`:

        $ cat > ~/bin/myssh
        #!/bin/sh
        # to enable X11 forwarding:
        exec ssh -X "$@"
        ^D

        $ chmod +x ~/bin/myssh

        $ export SSH_BATCH_SSH_CMD=~/bin/myssh
        $ atnodes 'ls -lh' '{my_cluster_name}'

    It's important to use "exec" in your own ssh wrapper script, or you may see `atnodes` hangs.

    This trick also works for the [key2nodes](https://metacpan.org/pod/key2nodes) script.

- Use wildcard for cluster expressions to save typing

    Wildcards in cluster spec could save a lot of typing. Say, if you have
    `api10.foo.bar.baz.bah.com.cn` appeared in your `~/.fornodesrc` file:

        $ cat ~/.fornodesrc
        MyCluster=api[01-22].foo.bar.baz.bah.com.cn

    then in case you want to refer to the `api10.foo.bar.baz.bah.com.cn` node alone on the command line, you can just say `api10*`, or `api10.*.com.cn`, or something more specific.

    But use wildcards with care. You may have nodes that you don't want in your
    resulting host list. So it's best practice to use [-l](https://metacpan.org/pod/-l) option when you use
    wildcards with [atnodes](https://metacpan.org/pod/atnodes) or [tonodes](https://metacpan.org/pod/tonodes), as in

        $ atnodes 'rm -rf /opt/blah' 'api10*' -l

    So that [atnodes](https://metacpan.org/pod/atnodes) will just echos out the exact host list that it would
    operate on but without doing anything. (It's effectively a "dry-run".)
    After checking, you can safely remove the `-l` option and go on.

- Specify a different ssh port or user name.

    You may have already learned that you can use the `-u` and `-p` options to specify a non-default user account or SSH port. But it's also possible and often more convenient to put it as part of your cluster spec expression, either in `~/.fornodesrc` or on the command line, as in

        $ cat > ~/.fornodesrc
        # cluster A uses the default user name:
        A=foo[01-25].com
        # cluster B uses the non-default user name "jim" and a port 12345
        B=jim@foo[26-28].com:12345

        $ atnodes 'ls -lh' '{B} + bob@bar[29-31].org:5678'

    It's also possible to specify a different rc config file than `~/.fornodesrc` via the environment variable `SSH_BATCH_RC`. For example,

        export SSH_BATCH_RC=/opt/my-fornodes-rc

    then the file `/opt/my-fornodes-rc` will be used instead of the default `~/.fornodesrc` file.

- Use `-L` to help grepping the outputs by hostname

    When managing hundreds or even thousands of machines, it's often more
    convenient to `grep` over the outputs of [atnodes](https://metacpan.org/pod/atnodes) or [tonodes](https://metacpan.org/pod/tonodes) by
    host names. The `-L` option makes [atnodes](https://metacpan.org/pod/atnodes) and [tonodes](https://metacpan.org/pod/tonodes) to prefixing
    every output lines of the remote commands (if any) by the host name. As in

        $ atnodes 'top -b|head -n5' '{my_big_cluster}' -L > out.txt 2>&1
        $ grep 'some.specific.host.com' out.txt

- Specify a timeout to prevent hanging

    It's often wise to specify a timeout for SSH operations. For example,
    if there's 3 sec of network traffic silence, the following command will
    quit with an error message printed:

        $ atnodes -t 3 'sleep 4' {my_cluster}

- Limit the bandwith used by [tonodes](https://metacpan.org/pod/tonodes) to be firewall-friendly

    You can use the `-b` option to tell [tonodes](https://metacpan.org/pod/tonodes) to use limited bandwidth
    if your intranet's Firewall is paranoid about your bandwidth use:

        $ tonodes my_big_file {my_cluster}:/tmp/ -b 8000

    where `8000` is in the unit of Kbits/sec, so it will not transfer
    faster than 1 MByte/sec.

- Avoid logging manually for the first time

    When you use [key2nodes](https://metacpan.org/pod/key2nodes) or [atnodes](https://metacpan.org/pod/atnodes) to access remote servers
    that you have never logged in manually, you would probably see the
    following errors:

        ===================== foo.com =====================
        Failed to spawn command.

        ERROR: unable to establish master SSH connection: the authenticity of the target host can't be established, try loging manually first

    A work-around is using "ssh" to login to that `foo.com` machine
    manually and then try [key2nodes](https://metacpan.org/pod/key2nodes) or [atnodes](https://metacpan.org/pod/atnodes) again.

    Another nicer work-around is to pass the `-o 'StrictHostKeyChecking=no'` option to the underlying `ssh` executable used by `SSH::Batch`.
    Here's a quick HOW-TO:

        $ cat > ~/bin/myssh
        #!/bin/sh
        # to disable StrictHostKeyChecking
        exec ssh -o 'StrictHostKeyChecking=no' "$@"
        ^D

        $ chmod +x ~/bin/myssh

        $ export SSH_BATCH_SSH_CMD=~/bin/myssh

        # then we try again
        $ key2nodes foo.com
        $ atnodes 'hostname' foo.com

[Back to TOC](#table-of-contents)

# PREREQUISITES

This module uses [Net::OpenSSH](https://metacpan.org/pod/Net::OpenSSH) behind the scene, so it requires the OpenSSH _client_ executable (usually spelled "ssh") with multiplexing support (at least OpenSSH 4.1). To check your `ssh` version, use the command:

    $ ssh -v

On my machine, it echos

    OpenSSH_4.7p1 Debian-8ubuntu1.2, OpenSSL 0.9.8g 19 Oct 2007
    usage: ssh [-1246AaCfgKkMNnqsTtVvXxY] [-b bind_address] [-c cipher_spec]
               [-D [bind_address:]port] [-e escape_char] [-F configfile]
               [-i identity_file] [-L [bind_address:]port:host:hostport]
               [-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] [-p port] [-R [bind_address:]port:host:hostport] [-S ctl_path]
               [-w local_tun[:remote_tun]] [user@]hostname [command]

There's no spesial requirement on the server side ssh service. Even a non-OpenSSH server-side deamon should work as well.

[Back to TOC](#table-of-contents)

# INSTALLATION

    perl Makefile.PL
    make
    make test
    sudo make install

Win32 users should replace "make" with "nmake".

[Back to TOC](#table-of-contents)

# CODE REPOSITORY

You can always get the latest `SSH::Batch` source from its public Git repository:

[http://github.com/agentzh/sshbatch](http://github.com/agentzh/sshbatch)

If you have a branch for me to pull, please let me know ;)

[Back to TOC](#table-of-contents)

# TODO

- Cache the parsing and evaluation results of the config file `~/.fornodesrc`
to somewhere like the fiel `~/.fornodesrc.cached`.
- Abstract the duplicate code found in the scripts to a shared .pm file.
- Add the `fromnodes` script to help downloading files from the remote
clusters to local file system (maybe grouped by host name).
- Add the `betweennodes` script to transfer files between clusters through
localhost.

[Back to TOC](#table-of-contents)

# AUTHORS

- Zhang "agentzh" Yichun (章亦春) `<agentzh@gmail.com>`
- Liseen Wan (万珣新) `<liseen.wan@gmail.com>`

[Back to TOC](#table-of-contents)

# COPYRIGHT & LICENSE

This module as well as its programs are licensed under the BSD License.

Copyright (c) 2009, Yahoo! China EEEE Works, Alibaba Inc. All rights reserved.

Copyright (C) 2009, 2010, 2011, Zhang "agentzh" Yichun (章亦春). All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

- Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
- Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
- Neither the name of the Yahoo! China EEEE Works, Alibaba Inc. nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[Back to TOC](#table-of-contents)

# SEE ALSO

[fornodes](https://metacpan.org/pod/fornodes), [atnodes](https://metacpan.org/pod/atnodes), [tonodes](https://metacpan.org/pod/tonodes), [key2nodes](https://metacpan.org/pod/key2nodes),
[SSH::Batch::ForNodes](https://metacpan.org/pod/SSH::Batch::ForNodes), [Net::OpenSSH](https://metacpan.org/pod/Net::OpenSSH).

[Back to TOC](#table-of-contents)

