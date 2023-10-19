# Mitmproxy & Redsocks

Guide for exploring balenaOS [Redsocks](https://github.com/darkk/redsocks) proxy usage with [Mitmproxy](https://mitmproxy.org/)

## Contents

- [Contents](#contents)
- [Requirements](#requirements)
- [Create Fleets](#create-fleets)
- [Proxy Device](#proxy-device)
- [Verify Proxy](#verify-proxy)
- [Store Mitmproxy Certificate](#store-mitmproxy-certificate)
- [Client Device](#client-device)
- [Captured Requests](#captured-requests)

## Requirements

Required is basic understanding on how to operate balena devices.

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

After provisioning and running the proxy device verify that the proxy is running by opening `http://192.168.111.36:8081/` in your browser. This would display the Mitmweb UI with no captured requests listed.

From your development machine or another device on the same network run `nslookup www.balena.io 192.168.111.36` to verify that the DNS forwarder is functioning properly.

## Store Mitmproxy Certificate

Open a terminal to the `proxy` container of the proxy device.

Run `cat ~/.mitmproxy/mitmproxy-ca-cert.pem | base64 -w 0` and store the produced string on your development machine. It is a long base64 single line string that ends with `=` (e.g. `LS0tLS1CRU ... UtLS0tLQo=`). This is the encoded self-generated certificate of Mitmproxy and will be used on the devices from the `proxy-client` fleet.

## Client Device

Provision a device with the `proxy-client` fleet created at the beginning. It does not have any pushed application code and containers to it yet as this is not needed yet.

Connect the client device to the same local network and start it.

## Captured Requests

![mitmweb](https://github.com/balena-io-experimental/proxy-mitm/assets/188837/9a305984-7495-4870-b98a-d9d8f439cbba)

