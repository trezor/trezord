# trezord

[![Build Status](https://travis-ci.org/trezor/trezord.svg?branch=master)](https://travis-ci.org/trezor/trezord) [![gitter](https://badges.gitter.im/trezor/community.svg)](https://gitter.im/trezor/community)

TREZOR Communication Daemon aka TREZOR Bridge

## What trezord does

`trezord` (short for TREZOR Daemon), or TREZOR Bridge, is a small piece of software, used for websites (such as our own [TREZOR Wallet](https://wallet.trezor.io)), to talk with TREZOR devices.

`trezord` starts a local webserver, with which both external applications and local applications can communicate. This webserver then communicates with TREZOR devices and returns their replies as JSON. `trezord` also manages access to devices - two applications cannot use the same device at the same time.

Communicating with devices using `trezord` is more high-level than communicating with devices directly - `trezord` abstracts away USB communication, Protobuf serialization and platform differences. However, you still need to process individual messages.

**For development of web apps for TREZOR, it is recommended to use [trezor.js](https://github.com/trezor/trezor.js) javascript API, which has separate javascript calls for most common usecases; or [TREZOR Connect](https://github.com/trezor/connect), which is even more high level.** (`trezor.js` communicates with `trezord` under the hood.)

## API documentation

`trezord` starts server on `localhost`, with port `21324`. You can use `https`, by using `https://localback.net:21324` which redirects to localhost. You can call this web address with standard AJAX calls from websites (see the note about whitelisting).

Server supports following API calls:

| url <br> method | parameters | result type | description |
|-------------|------------|-------------|-------------|
| `/` <br> GET | | {`version`:&nbsp;string,<br> `configured`:&nbsp;boolean,<br> `validUntil`:&nbsp;timestamp} | Returns current version of bridge and info about configuration.<br>See `/configure` for more info. |
| `/configure` <br> POST | request body: config, as hex string | {} | Before any advanced call, configuration file needs to be loaded to bridge.<br> Configuration file is signed by SatoshiLabs and the validity of the signature is limited.<br>Current config should be [in this repo](https://github.com/trezor/webwallet-data/blob/master/config_signed.bin), or [on AWS here](https://wallet.trezor.io/data/config_signed.bin). |
| `/enumerate` <br> GET | | Array&lt;{`path`:&nbsp;string, <br>`session`:&nbsp;string&nbsp;&#124;&nbsp;null}&gt; | Lists devices.<br>`path` uniquely defines device between more connected devices. It might or might not be unique over time; on some platform it changes, on others given USB port always returns the same path.<br>If `session` is null, nobody else is using the device; if it's string, it identifies who is using it. |
| `/listen` <br> POST | request body: previous, as JSON | like `enumerate` | Listen to changes and returns either on change or after 30 second timeout. Compares change from `previous` that is sent as a parameter. "Change" is both connecting/disconnecting and session change. |
| `/acquire/PATH/PREVIOUS` <br> POST | `PATH`: path of device<br>`PREVNOUS`: previous session (or string "null") | {`session`:&nbsp;string} | Acquires the device at `PATH`. By "acquiring" the device, you are claiming the device for yourself.<br>Before acquiring, checks that the current session is `PREVIOUS`.<br>If two applications call `acquire` on a newly connected device at the same time, only one of them succeed. |
| `/release/SESSION`<br>POST | `SESSION`: session to release | {} | Releases the device with the given session.<br>By "releasing" the device, you claim that you don't want to use the device anymore. |
| `/call/SESSION`<br>POST | `SESSION`: session to call<br><br>request body: JSON <br>{`type`: string, `message`: object}  | {`type`: string, `body`: object} | Calls the message and returns the response from TREZOR.<br>Messages are defined in [this protobuf file](https://github.com/trezor/trezor-common/blob/master/protob/messages.proto).<br>`type` in request is, for example, `GetFeatures`; `type` in response is, for example, `Features` |

### Whitelisting

You cannot connect to `trezord` from anywhere on the internet. Your URL needs to be specifically whitelisted; whitelist is in the signed config file, that is sent during `configure/` call.

`localhost` is specifically whitelisted, so you can experiment on `http://localhost`. If you want to add your url in order to make a TREZOR web app, [make a pull request to this file](https://github.com/trezor/trezor-common/blob/master/signer/config.json).

## Download latest binary

Latest build packages are on https://wallet.trezor.io/data/bridge/latest/index.html

## Checking out sources

```
git clone https://github.com/trezor/trezord.git
cd trezord
git submodule update --init
```

## Building

Change into `release/linux` or `release/windows` directory and run: `make`.

Or run `build.sh` to build locally using local dependencies.

In the latter case, here is a rough list of Debian package build dependencies:

`build-essential cmake curl libcurl4-gnutls-dev libprotobuf-dev pkg-config libusb-1.0-0 libusb-1.0-0-dev libmicrohttpd-dev libboost-all-dev protobuf-compiler`

or Fedora (as of 26) build dependencies:

`boost-devel-static protobuf-compiler cmake gcc-c++ libcurl-devel protobuf-devel libusbx-devel libmicrohttpd-devel protobuf-static`

Also you might need to regenerate protobuf files if you are using protobuf-3.x:

```
cd src/config
wget https://raw.githubusercontent.com/trezor/trezor-common/master/protob/config.proto
protoc -I/usr/include -I. --cpp_out=. config.proto
```
