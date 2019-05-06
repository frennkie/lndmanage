lndmanage
---------

lndmanage is a command line tool for advanced channel management of an [`LND`](https://github.com/lightningnetwork/lnd) node.

Current feature list (use the ```--help``` flag for subcommands):

* advanced node summary ```status```
* compact ```listchannels``` commands:
  * list channels for rebalancing ```listchannels rebalance```
  * list inactive channels for channel hygiene ```listchannels inactive```
  * list forwarding statistics for each channel ```listchannels forwardings```
* rebalancing of channels ```rebalance```
  * different strategies can be chosen
  * a target 'balancedness' can be specified (e.g. to empty the channel)
* doing circular self-payments ```circle```

**DISCLAIMER: This is BETA software, so please be careful (All actions are executed as a dry run unless you call lndmanage with the ```--reckless``` flag though). No warranty is given.**

Command line options
--------------------
```sh
usage: lndmanage.py [-h] [--loglevel {INFO,DEBUG}]
                    {status,listchannels,rebalance,circle} ...

Lightning network daemon channel management tool.

positional arguments:
  {status,listchannels,rebalance,circle}
    status              display node status
    listchannels        lists channels with extended information [see also
                        subcommands with -h]
    rebalance           rebalance a channel
    circle              circular self-payment

```

Rebalancing a channel
---------------------
The workflow for rebalancing a channel goes as follows:

* take a look at all your unbalanced channels with:

  ```$ ./lndmanage.py listchannels rebalance```
  
    The output will look like:
  ```
  -------- Description --------
  ub         unbalancedness (see --help)
  cap        channel capacity [sats]
  lb         local balance [sats]
  rb         remote balance [sats]
  bf         peer base fee [msats]
  fr         peer fee rate
  cid        channel id
  a          alias
  
  -------- Channels --------
         cid            ub       cap        lb        rb     bf        fr  a       
  xxxxxxxxxxxxxxxxxx -0.78   1000000    888861     99480     10  0.000200 abc                
  xxxxxxxxxxxxxxxxxx -0.63   1000000    814537    173768    300  0.000010 def
  xxxxxxxxxxxxxxxxxx  0.55   2000000    450792   1540038     35  0.000002 ghi
  xxxxxxxxxxxxxxxxxx  0.59    400000     81971    306335    400  0.000101 jkl
  ...
  ```
* the ```ub``` field tells you how unbalanced your channel is and in which direction
* take a channel_id from the list you wish to rebalance (target is a 50:50 balance)
* do a dry run to see what's waiting for you

  ```$ ./lndmange.py rebalance --max-fee-sat 20 --max-fee-rate 0.00001 channel_id```

* read the output and if everything looks well, then run with the ```--reckless``` flag

Doing channel hygiene
---------------------
Inactive channels lock up capital, which can be used elsewhere. In order to close those channels it is useful to take a look
at the inactive channels with ```$ ./lndmanage.py listchannels inactive```.

You will get an output like:

```sh
-------- Description --------
p          true if private channel
ini        true if we opened channel
lup        last update time [days ago]
age        channel age [days]
cap        capacity [sats]
lb         local balance [sats]
sr/w       satoshis sent + received per week of lifespan
cid        channel id
a          alias

-------- Channels --------
       cid         p ini   lup   age       cap        lb     sr/w  a       
xxxxxxxxxxxxxxxxxx F   F    66    71   2000000     10000      100 abc
xxxxxxxxxxxxxxxxxx T   F    20   113     40000         0        0 def
xxxxxxxxxxxxxxxxxx F   T    19    21   1200000      1000        0 ghi
...
```
Channels, which were updated a long time ago (```lup```) are likely to be inactive in the future and may be closed.

Another way to see if funds have to be reallocated is to have a look at
the forwarding statistics of, e.g., the last two months of the individual channels
 with ```$./lndmanage.py listchannels forwardings --from-days-ago 60 --sort-by='fees'```
 (here sorted by total fees, but it can be sorted by any column field).

The output will look like:
```
-------- Description --------
cid        channel id
nfwd       number of forwardings
age        channel age [days]
fees       fees total [sats]
f/w        fees per week [sats]
ub         unbalancedness
flow       flow direction (positive is outwards)
r          rebalance required if marked with X
cap        channel capacity [sats]
in         total forwardings inwards [sats]
imed       median forwarding inwards [sats]
imax       largest forwarding inwards [sats]
out        total forwardings outwards [sats]
omed       median forwarding outwards [sats]
omax       largest forwarding outwards [sats]
a          alias

-------- Channels --------
       cid         nfwd   age  fees     f/w    ub  flow r      cap         in    imed    imax        out    omed    omax    a
xxxxxxxxxxxxxxxxxx    6   103   907 106.950  0.30  1.00 X  6000000 |        0     nan     nan |  1935309   20000 1800902 | abc
xxxxxxxxxxxxxxxxxx    3    82   300  35.374  0.74 -0.08    1000000 |   700008  700008  700008 |   600000  600000  600000 | def
xxxxxxxxxxxxxxxxxx    4    32   216  25.461  0.38  0.42 X  6000000 |   993591  993591  993591 |  2450000  750000 1000000 | ghi
...
```

Setup
-----
Requirements: python3.6, lnd 0.6
```
$ virtualenv -p python3 ~/.venvs/lndmanage
$ source ~/.venvs/lndmanage/bin/activate
$ git clone https://github.com/bitromortac/lndmanage
$ cd lndmanage
$ pip install -r requirements.txt
$ cp config_sample.ini config.ini
```

Edit configuration (config.ini):
* lnd_grpc_host: ip and port of the grpc API
* tls_cert_file: location of the tls certificate
* admin_macaroon_file: location of the admin macaroon

Test:
```
$ ./lndmanage.py status 
```
If if works, you should see the node status.

Docker
------

If you prefer to run `lndmanage` from a docker container, `cd docker` and follow [`README`](docker/README.md) there.

Compiling grpc in python [development]
----------------------------------------------------
```
$ cd grpc_compile
$ pip install grpcio grpcio-tools googleapis-common-protos
$ git clone https://github.com/googleapis/googleapis.git
$ curl -o rpc.proto -s https://raw.githubusercontent.com/lightningnetwork/lnd/master/lnrpc/rpc.proto
$ python -m grpc_tools.protoc --proto_path=googleapis:. --python_out=. --grpc_python_out=. rpc.proto
```
