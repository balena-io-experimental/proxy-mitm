# Mitmproxy & Redsocks

Step-by-step guide for exploring balenaOS [Redsocks](https://github.com/darkk/redsocks) proxy usage with [Mitmproxy](https://mitmproxy.org/)

## Contents

- [Contents](#contents)
- [Requirements](#requirements)
- [Create Fleets](#create-fleets)
- [Proxy Device](#proxy-device)
- [Verify Proxy](#verify-proxy)
- [Store Mitmproxy Certificate](#store-mitmproxy-certificate)
- [Client Device](#client-device)
- [Configure config.json](#configure-configjson)
- [Configure Redsocks](#configure-redsocks)
- [Captured Requests](#captured-requests)
- [Redsocks Iptables](#redsocks-iptables)
- [Excluding IP Addresses](#excluding-ip-addresses)

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

From the balena dashboard take note of the local IP address of the device. In this README we will be using the `192.168.111.36` IP address, but that would be different in your setup.

## Verify Proxy

After provisioning and running the proxy device verify that the proxy is running by opening `http://192.168.111.36:8081/` in your browser. This would display the Mitmweb UI with no captured requests listed.

From your development machine or another device on the same network run `nslookup www.balena.io 192.168.111.36` to verify that the DNS forwarder is functioning properly.

## Store Mitmproxy Certificate

Open a terminal to the `proxy` container of the proxy device.

Run `cat ~/.mitmproxy/mitmproxy-ca-cert.pem | base64 -w 0` and store the produced string on your development machine. It is a long base64 single line string that ends with `=` (e.g. `LS0tLS1CRU ... UtLS0tLQo=`). This is the encoded self-generated certificate of Mitmproxy and will be used on the devices from the `proxy-client` fleet.

## Client Device

Provision a device with the `proxy-client` fleet created at the beginning. You do not need to push any application code and containers to it yet.

Connect the client device to the same local network and start it.

## Configure config.json

Add `"dnsServers": "192.168.111.36"` entry to the config.json to use the DNS forwarder on the proxy device. Depending on how you have configured your local network you may not need it at all, but it is added for convenience.

Add `"balenaRootCA": "LS0tLS1CRU ... UtLS0tLQo="` entry with the saved encoded Mitmproxy certificate. This is needed by the proxy in order to decode HTTPS content. In a normal proxy setting this won't be usually needed, but it is another huge convenience for debugging and exploration purposes.

## Configure Redsocks

Open a host OS terminal to the client device.

Populate the `/mnt/boot/system-proxy/redsocks.conf.ignore` file with the following content:

```
base {
  log_debug = on;
  log_info = on;
  log = stderr;
  daemon = off;
  redirector = iptables;
}

redsocks {
  type = http-connect;
  ip = 192.168.111.36;
  port = 8080;
  local_ip = 127.0.0.1;
  local_port = 12345;
}
```

As usual replace the `192.168.111.36` address with the local address of the proxy device.

Run `mv /mnt/boot/system-proxy/redsocks.conf.ignore /mnt/boot/system-proxy/redsocks.conf` in order to enable the default redsocks configuration.

Reboot the device.

## Captured Requests

After the device is rebooted you should see the client device traffic going through the Mitmproxy:

![mitmweb](https://github.com/balena-io-experimental/proxy-mitm/assets/188837/9a305984-7495-4870-b98a-d9d8f439cbba)

Note: The unencoded TCP traffic in the above screenshot is the traffic from the OpenVPN client. 

## Redsocks Iptables

In order for Redsocks to capture and redirect traffic from the client device to the proxy it needs a set of iptables rules added. It is out of the scope of this document to describe those in details, but the Redsocks README contains such [short description](https://github.com/darkk/redsocks#iptables-example).

Under balenaOS those rules are added automatically. You may check them on a client device by running `iptables -t nat -L -n`.

Note the `10.0.0.0/8`, `100.64.0.0/10`, `169.254.0.0/16` and other ranges in the REDSOCKS chain. Those are private IP address ranges and as it should be expected connections to such addresses will not be redirected to the proxy. In the next section the same technique will be used to exclude Internet public IP addresses.

## Excluding IP Addresses

If you would like to exclude certain IP addresses from the Redsocks configuration so that your device reaches them directly, you need to modify the REDSOCKS nat chain.

First clear all the captured requests from the Mitmproxy UI (File/Clear all).

From a host OS terminal on the client device run `curl https://1.1.1.1/`. Now check the Mitmweb UI and you should see the proxied decoded request to `https://1.1.1.1/`.

Next exlude the `1.1.1.1` address from Redsocks by running `iptables -t nat -I REDSOCKS 1 -d 1.1.1.1 -j RETURN`. You may check the REDSOCKS chain to see the newly added rule with `iptables -t nat -L -n`.

Clear all captured requests and run again `curl https://1.1.1.1/`. Now the request goes directly to the Cloudflare server and does not pass through the proxy. You can verify that by checking the Mitmweb UI.

Iptables rules added through a host OS terminal will not be persisted upon reboots and for production usage those should be added from a container. Alternatively those can be added with [NetworkManager User Scripts](https://docs.balena.io/reference/OS/network/#networkmanager-user-scripts) as well.
