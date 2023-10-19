# Mitmproxy & Redsocks

Setup guide for exploring balenaOS proxy settings using Mitmproxy

## Contents

- [Contents](#contents)
- [Create Fleets](#create-fleets)
- [Proxy Device](#proxy-device)
- [Mitmweb](#mitmweb)

## Create Fleets

Create two separate balena fleets called `proxy-mitm` and `host`.

The `proxy-mitm` fleet will have one device only that runs a proxy service and a DNS forwarder.

The `host` fleet could be used by multiple devices that have their redsocks proxy configured.

## Proxy Device

Provision one device using this repository. This could be a physical balena device or a virtual device.

One good option is using Virtual Machine Manager (virt-manager) with a "Generic x86_64 (GPT)" image, but please use device type that is most convenient for you.

The device should be easily accessible through a local physical or virtual network. 

## Mitmweb Web Interface

![mitmweb](https://github.com/balena-io-experimental/proxy-mitm/assets/188837/9a305984-7495-4870-b98a-d9d8f439cbba)

