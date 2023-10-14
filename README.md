# Wireguard VPN Server Setup Playbook

This Ansible playbook will provision a Debian 11 VM/VPS to act as an ephemeral VPN using Wireguard.

## Usage

First, create a VM/VPS instance running Debian 11 and get the credentials necessary to run as 'root'. Then, run a command like this where `123.234.321.4` is the public IP address of your VM/VPS:

```sh
$ ansible-playbook -i 123.234.321.4, -u root wg_server.yml
```

This will produce some output. The most important lines are these:
```
TASK [debug] *******************************************************************
ok: [2001:19f0:5c00:10d2:5400:04ff:fe2e:b2fd] => {
    "msg": "Client Private Key: UJ5jY2NC/xrW3dhdgInkVRUbFwxHLfRm4MEMPtUr5l0="
}

TASK [debug] *******************************************************************
ok: [2001:19f0:5c00:10d2:5400:04ff:fe2e:b2fd] => {
    "msg": "Server Public Key: 0McmAQdUAYsG6+kClrAdyR4S2NMF1ChLXrJbVHfiwi4="
```

Once the playbook has finished successfully, it's time to create the new Wireguard interface. First, you need to write the Client Private Key to a file (due to annoying behavior in `wg`):
```sh
$ echo 'UJ5jY2NC/xrW3dhdgInkVRUbFwxHLfRm4MEMPtUr5l0=' > /tmp/privkey
```

Then, you can create a Wireguard network interface and move it into a network namespace:
```sh
$ sudo mkdir -p /etc/netns/VPN/
$ echo 'nameserver 1.1.1.1' | sudo tee /etc/netns/VPN/resolv.conf
$ sudo ip link add wg0 type wireguard
$ sudo ip netns add VPN
$ sudo ip link set wg0 netns VPN
```

Finally, you can configure the Wireguard interface, where `0McmAQdUAYsG6+kClrAdyR4S2NMF1ChLXrJbVHfiwi4=` is the peer's public key as printed in the playbook execution output:
```sh
$ sudo ip -n VPN link set wg0 up
$ sudo ip netns exec VPN wg set wg0 private-key /tmp/privkey peer 0McmAQdUAYsG6+kClrAdyR4S2NMF1ChLXrJbVHfiwi4= endpoint 123.234.321.4:1198 allowed-ips 0.0.0.0/0
$ sudo ip -n VPN addr add 10.1.1.1/30 dev wg0
$ sudo ip -n VPN route add default via 10.1.1.2
```

Now all you need to do to run traffic through the VPN is execute your programs in the newly-created `VPN` network namespace!
```bash
$ sudo ip netns exec VPN runuser $USER
$ # inside network namespace
$ curl icanhazip.com
$ firefox --no-remote -p
```
