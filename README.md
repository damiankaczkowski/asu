# Attendedsysupgrade Server (GSoC 2017)

This project intend to simplify the sysupgrade process of LEDE/LibreMesh. The provided tools here offer an easy way to reflash the router with a new release or package updates, without the need of `opkg` installed.

![luci-app-attendedsysupgrade-screenshot](https://camo.githubusercontent.com/d21d3c2e43993325c0371866b28f09a67ea21902/687474703a2f2f692e696d6775722e636f6d2f653443716841502e706e67)

## Created LEDE packages

#### [`luci-app-attendedsysupgrade`](https://github.com/openwrt/luci/tree/master/applications/luci-app-attendedsysupgrade)

Add a view to the Luci system tab called "Attended Sysupgrade". Offers a button to search for updates and if found, to flash the image created by the update server.

**Dependencies:**
* `rpcd-mod-rpcsys`
	Used to read list of installed packages and trigger sysupgrade on the target.
* `uhttpd-mod-ubus`
	Communication between the Browser and the Router
* `cgi-io`
	Upload image from Browser to Router

#### [`auc`](https://github.com/openwrt/packages/tree/master/utils/auc)

Add CLI to perform sysupgrades.

* `rpcd-mod-rpcsys`
	Used to read list of installed packages and trigger sysupgrade on the target.
* `usteam-ssl` and `ca-certificates`
	Securely communicate and download firmware from server via https

## Server side

The server listens to update and image requests. Images are auto generated if the requests was valid. LEDE ImageBuilder is automatically setup up on first request of distribution, release, target & subtarget.

All requests are stored in a queue and build by workers.

### Active server

##### [planetexpress](https://ledeupdate.planetexpress.cc) - thanks @egon0

## API

To communicate with the update server one have to distinguish between *upgrade check* and *upgrade request*. Both are explained below.

### Upgrade check `/api/upgrade-check`

Sends information about the device to the server to see if a new distribution release or package upgrades are available. An *upgrade check* could look like this:

| key	| value | information	|
| ---	| ---	| ---		|
| `distro` | `LEDE` | installed distribution |
| `version` | `17.01.0` | installed release |
| `target` | `ar71xx` | |
| `subtarget` | `generic` | |
| `packages` | `{libuci-lua: 2017-04-12-c4df32b3-1, cgi-io: 3, ...}` | all user installed packages |

Most information can be retrieved via `ubus call system board`. Missing information can be gathered via the `rpcd-mod-packagelist` package.
`packages` contains all user installed packages. Packages installed as an dependence are excluded.

It's also possible to check for a new release without sending packages by removing `packages` from the request.

### Response `status 200`

The server validates the request. Below is a possible response for a new release:

| key		| value		| information	|
| ---		| ---		| ---		|
| `version`		| `17.01.2`		| newest release |
| `updates`		| `{luci-lib-jsonc: [git-17.230.25723-2163284-1, git-17.228.56579-209deb5-1], ...}` | Package updates `[new_version, current_version]` |
| `packages`	| `[libuci-lua, cgi-io: 3, ...]` | All packages for the new image |

See [other status codes](#response-status-codes)

An release upgrade does not ignore package upgrades for the following reason. Between releases it possible that package names may change, packages are dropped or merged. The *response* contains all packages included changed ones.

### Upgrade request `/api/upgrade-request`

The *upgrade check response* should be shown to the user in a readable way. Once the user decides to perform the sysupgrade a new request is send to the server called *upgrade request*

| key		| value					| information	|
| ---		| ---					| ---		|
| `distro`	| `LEDE`				| installed distribution |
| `version`	| `17.01.2`					| installed release |
| `target`	| `ar71xx`				| |
| `subtarget`	| `generic`					| |
| `board`	| `tl-wdr4300-v1`			| `board_name` of `ubus call system board` |
| `[model]`		| `TP-Link TL-WDR4300 v1`		| `model` of `ubus call system board`. This is optional and a fallback |
| `packages`	| `[libuci-lua, cgi-io: 3, ...]`	| All packages for the new image |

The *upgrade request* is nearly the same as the *upgrade check* before, except only containing package names without version and adding `board` and possibly `model`. While the server builds the requested image the clients keeps polling the server, sending *exacly* the same *image request*. The client _does not_ receive a ticket ID or anything similar. This is due to the situation when same devices poll in parallel only one build job is triggered, resulting in the same image for all devices.

### Response `status 200`

| key	| value | information	|
| ---	| ---	| ---		|
| `checksum` | `8b32f569dca8dcd780414f7850ed11e8` | md5 |
| `filesize` | `5570113` |  |
| `sysupgrade` | `https://betaupdate.libremesh.o…x86-64-Generic-sysupgrade.bin` | download link |

See [other status codes](#response-status-codes)

### Build request `/api/build-request`

It's also possible to request to build an image. The request is exactly the same as for `upgrade-request`. The response only contains a link to the created `files` or `upgrade-request` parameters if available.

#### Response `status 200`

| key		| value		| information	|
| ---		| ---		| ---		|
| `files`	| `https://betaupdate.libremesh.o...neric/af937867acd2951/`		| files in json format |

See [other status codes](#response-status-codes)

This can be used to create a service like the *[Online Imagebuilder](https://chef.libremesh.org)*

### Response status codes

The client should check the status code:

| status	| meaning												| information	|
| ---		| ---								| ---			|
| 200		| build finish / upgrade available	| see parameters of `upgrade-check`, `upgrade-request` or `build-request` |
| 202		| building, queued, imagebuilder setup	| building right now, in build queue, imagebuilder not ready. Details are in header `X-Imagebuilder-Status` and `X-Build-Queue-Position` |
| 204		| no updates						| device is up to date |
| 400		| bad request 						| see `error` parameter |
| 413		| imagesize fail					| produced image to big for device |
| 413		| imagesize fail					| produced image to big for device |
| 422		| unknown package					| unknown package in request |
| 500		| build failed						| see `log` for build log	|
| 501		| no sysupgrade						| image build successful but no sysupgrade image created |
| 502		| proxy backend down				| nginx runs but python part is down, likely maintenance |
| 503		| server overload					| please wait ~5 minutes |
