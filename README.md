# Mitmproxy & Redsocks

Setup guide for exploring balenaOS proxy settings using Mitmproxy

## Contents

- [Requirements](#requirements)
- [Contents](#contents)
- [Create Fleets](#create-fleets)
- [Proxy Device](#proxy-device)
- [Verify Proxy](#verify-proxy)
- [Mitmweb](#mitmweb)

## Requirements

You need some understanding on how to operate balena devices. This is not a [getting started guide](https://docs.balena.io/learn/getting-started). 

## Create Fleets

Create two separate balena fleets called `proxy-mitm` and `proxy-client`.

The `proxy-mitm` fleet will have one device only that runs a proxy service and a DNS forwarder.

The `proxy-client` fleet could be used by multiple devices that have their Redsocks proxy configured.

## Proxy Device

Push this repository towards the `proxy-mitm` fleet and provision the proxy device with it. This could be a physical balena device or a virtual device.

One good option is using Virtual Machine Manager (virt-manager) with a "Generic x86_64 (GPT)" image, but please use device type that is most convenient for you.

The device should be easily accessible through a local physical or virtual network.

Since the device will be used for exploration and debugging purposes, it is a good practice to provision it in [development mode](https://docs.balena.io/reference/OS/configuration/#developmentmode).

Connect the proxy device to a local network and start it.

From the balena dashboard take note of the local IP address of the device. In this README we will be using a `192.168.111.36` IP address, but that would be different in your setup.

## Verify Proxy

After provisioning and running the proxy device verify that the proxy is running by opening `http://192.168.111.36:8081/` in your browser.

From your development machine run `nslookup www.balena.io 192.168.111.36` to verify that the DNS forwarder is functioning properly.

## Mitmweb Web Interface

![mitmweb](https://github.com/balena-io-experimental/proxy-mitm/assets/188837/9a305984-7495-4870-b98a-d9d8f439cbba)

