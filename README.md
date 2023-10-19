# Mitmproxy & Redsocks

Step-by-step guide to explore balenaOS [Redsocks](https://github.com/darkk/redsocks) proxy usage with [Mitmproxy](https://mitmproxy.org/)

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

Basic understanding of how to operate balena devices is required.

## Create Fleets

Create two distinct balena fleets named 'proxy-mitm' and 'proxy-client'.

The 'proxy-mitm' fleet will consist of only one device responsible for operating a proxy service and a DNS forwarder.

The 'proxy-client' fleet can accommodate multiple devices with the same Redsocks proxy configuration.

## Proxy Device

Push this repository to the 'proxy-mitm' fleet and provision the proxy device with it. The device can be a physical balena device or a virtual device.

One good option is to use Virtual Machine Manager (virt-manager) with a "Generic x86_64 (GPT)" image, but please select the device type that is most convenient for you.

Ensure that the device is easily accessible through a local physical or virtual network.

Since the device will be used for exploration and debugging purposes, it is a good practice to provision it in [development mode](https://docs.balena.io/reference/OS/configuration/#developmentmode).

Connect the proxy device to a local network and start it.

From the balena dashboard, take note of the local IP address of the device. In this README, we will be using the `192.168.111.36` IP address, but it may differ in your setup.

## Verify Proxy

After provisioning and running the proxy device, verify that the proxy is operational by opening `http://192.168.111.36:8081/` in your browser. This should display the Mitmweb UI with no captured requests listed.

From your development machine or another device on the same network, execute the command `nslookup www.balena.io 192.168.111.36` to confirm the proper functioning of the DNS forwarder.

## Store Mitmproxy Certificate

Open a terminal to the 'proxy' container of the proxy device.

Execute the command `cat ~/.mitmproxy/mitmproxy-ca-cert.pem | base64 -w 0` and store the resulting string on your development computer. It is a lengthy base64 single line string that concludes with `=` (e.g., `LS0tLS1CRU ... UtLS0tLQo=`). This encoded string is the self-generated Mitmproxy certificate and will be utilized on the devices from the 'proxy-client' fleet.

## Client Device

Provision a device with the 'proxy-client' fleet created at the beginning. You do not need to push any application code or containers to it at this stage.

Connect the client device to the same local network and start it.

## Configure config.json

Add the entry `"dnsServers": "192.168.111.36"` to the config.json to utilize the DNS forwarder on the proxy device. Depending on your local network configuration, you may not need this entry, but it is included for convenience.

Include the entry `"balenaRootCA": "LS0tLS1CRU ... UtLS0tLQo="` with the saved encoded Mitmproxy certificate. This is necessary for the proxy to decode HTTPS content. While this may not typically be required in a regular proxy setup, it is needed here for debugging and exploration purposes.

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

As usual, replace the `192.168.111.36` address with the local address of the proxy device.

Execute the command `mv /mnt/boot/system-proxy/redsocks.conf.ignore /mnt/boot/system-proxy/redsocks.conf` to activate the new Redsocks configuration.

Reboot the device.

## Captured Requests

After the device is rebooted you should see the client device traffic going through the Mitmproxy:

![mitmweb](https://github.com/balena-io-experimental/proxy-mitm/assets/188837/9a305984-7495-4870-b98a-d9d8f439cbba)

Note: The unencoded TCP traffic in the above screenshot is the traffic from the OpenVPN client. 

## Redsocks Iptables

In order for Redsocks to capture and redirect traffic from the client device to the proxy, it requires a set of iptables rules. While it is beyond the scope of this document to describe them in detail, the Redsocks README provides a brief description, which can be found [here](https://github.com/darkk/redsocks#iptables-example).

Under balenaOS, these rules are automatically added. You can verify them on a client device by running `iptables -t nat -L -n`.

Take notice of the `10.0.0.0/8`, `100.64.0.0/10`, and `169.254.0.0/16` ranges in the REDSOCKS chain. These are private IP address ranges, and as expected, connections to such addresses should not be redirected to the proxy. The next section will use the same technique to exclude public Internet IP addresses.

## Excluding IP Addresses

If you wish to exclude specific IP addresses from the Redsocks configuration, allowing your device to reach them directly, you'll need to modify the REDSOCKS NAT chain.

Start by clearing all the captured requests from the Mitmproxy UI (select File, then Clear all).

From a host OS terminal on the client device, execute `curl https://1.1.1.1/`. Check the Mitmweb UI to confirm that the proxied decoded request to `https://1.1.1.1/` is visible.

Next, exclude the `1.1.1.1` address from Redsocks by running `iptables -t nat -I REDSOCKS 1 -d 1.1.1.1 -j RETURN`. You can check the REDSOCKS chain to view the newly added rule at the top by executing `iptables -t nat -L -n`.

Clear all captured requests and run `curl https://1.1.1.1/` again. This time, the request will go directly to the '1,1,1,1' Cloudflare server and will not pass through the proxy. You can verify this by checking the Mitmweb UI.

Please note that iptables rules added through a host OS terminal will not persist upon reboots. For production usage, these rules should be added from a container. Alternatively, they can be added through [NetworkManager User Scripts](https://docs.balena.io/reference/OS/network/#networkmanager-user-scripts) as well.

## Going to Production

The Mitmproxy solution is primarily useful for exploration and debugging purposes. For production usage, we do not have any official recommendations, but you might consider a well-established proxy server such as Squid Cache or Nginx Proxy.
