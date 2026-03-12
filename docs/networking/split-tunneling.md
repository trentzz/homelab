# Split Tunneling with a VPN

Suppose we want to use a VPN to access something specific, e.g. using a mesh VPN like netbird to access another computer, or Mullvad/NordVPN or a host of others to spoof location. We run into an issue if we also want to access internal IP addresses, like nfs shares.

Enter split tunneling. With this we can specify that we want to bypass the VPN when accessing a specific range of IP addresses (e.g. your home network).

Search up how to do this with your VPN client, most should have easy instructions on how to do this.
