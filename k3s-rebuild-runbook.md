# k3s Server Rebuild Runbook (Alpine)

Rebuild steps for `server.hubble`, worked out from the root and `fredo` shell histories after the kernel vulnerability rebuild. Where the reason for a step had to be guessed, it is marked *(likely)*.

---

## 1. Base networking and packages

1. Take wifi down so the box runs on wired only:
   ```sh
   ip link set wlan0 down
   ifdown wlan0
   ```
2. Fix DNS in `/etc/resolv.conf`. `apk` won't work until this is done. > This was because the DHCP dns runs on the host.
3. Edit the network config, then reboot:
   - `/etc/network/interfaces`
   - `/etc/udhcpc/udhcpc.conf`
   - `/etc/network/if-up.d/dad`
4. Stop wifi coming back on boot:
   ```sh
   rc-update del wpa_supplicant boot
   ```
5. Update packages and enable the community repo (ufw needs it):
   ```sh
   apk update
   setup-apkrepos
   ```

## 2. User and privilege escalation

6. Add the user to the wheel group:
   ```sh
   adduser fredo wheel
   ```
7. Configure doas in `/etc/doas.d/20-wheel.conf`:
   ```
   permit persist :wheel
   ```

## 3. NFS mount

8. Create the mount point and mount the share:
   ```sh
   mkdir /mnt/hubble
   mount -t nfs -o nolock 192.168.0.x:/nfs/Hubble /mnt/hubble
   ```
9. Append the backed-up `fstab` from the share to `/etc/fstab` and tidy it up:
   ```sh
   cat /mnt/hubble/fstab >> /etc/fstab
   vi /etc/fstab
   mount -a
   ```
10. Reboot and check that the mount comes back (`df -h`).

## 4. Firewall

11. Install ufw:
    ```sh
    apk add ip6tables ufw
    ```
12. Add the rules:
    ```sh
    ufw allow 6443/tcp                   # k3s API
    ufw allow from 10.42.0.0/16 to any   # pod CIDR
    ufw allow from 10.43.0.0/16 to any   # service CIDR
    ufw allow ssh/tcp
    ufw enable
    ```
13. Start the service and make it start on boot:
    ```sh
    rc-service ufw start
    rc-update add ufw default
    ```
    > `rc-service ufw enable` (from the history) isn't a valid OpenRC command. Use `rc-update add`.

## 5. Kernel prep and k3s install

14. Check cgroups (`/sys/fs/cgroup/memory*`) and edit `/boot/extlinux.conf`. *(Likely: this added the cgroup kernel args such as `cgroup_enable=memory`.)* Then reboot.
15. Install k3s:
    ```sh
    apk add curl
    curl -sfL https://get.k3s.io | sh -
    ```
16. Check that it is up:
    ```sh
    k3s kubectl get nodes
    rc-update | grep k3s   # confirm it starts on boot
    ```

## 6. Kubeconfig for fredo

17. Give fredo his own copy of the kubeconfig:
    ```sh
    doas cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
    doas chown fredo ~/.kube/config
    export KUBECONFIG=~/.kube/config
    ```
    > Add the `export` to your shell profile so it persists.

## 7. Restore cluster state from backup

> The token, the datastore and the `tls/` directory **must all come from the same backup**, or k3s won't start cleanly.

18. Copy the backups into place:
    ```sh
    cd /var/lib/rancher/k3s
    cp /mnt/hubble/etcd.tar.gz /mnt/hubble/rancher-token /mnt/hubble/k3s-pvcs.tar.gz .
    ```
19. Stop k3s:
    ```sh
    rc-service k3s stop
    ```
20. **Datastore:**
    ```sh
    mv etcd.tar.gz server/
    cd server
    mv db db.bak
    tar xvf etcd.tar.gz
    mv var/lib/rancher/k3s/server/db .
    rm -rf var/ etcd.tar.gz
    cd ..
    ```
21. **Token:**
    ```sh
    mv rancher-token server/token
    chmod 600 server/token
    ```
22. **PVC data** (still in `/var/lib/rancher/k3s`):
    ```sh
    tar xvf k3s-pvcs.tar.gz
    rm -f k3s-pvcs.tar.gz
    ```
23. **Server backup and manifests:**
    ```sh
    cd /mnt/hubble
    tar xvf server.tar.gz
    cp -rp server/manifests/nfs.yaml /var/lib/rancher/k3s/server/manifests/
    ```
24. **TLS.** Restore the old certs so the CA matches the restored datastore:
    ```sh
    cd /var/lib/rancher/k3s/server
    mv tls tls.bak
    mv /mnt/hubble/server/tls tls
    ```
25. Start k3s and check the pods:
    ```sh
    rc-service k3s start
    rc-service k3s restart   # if needed
    kubectl get pods -A
    ```
    Then reboot, check again, and look the cluster over in `k9s`.

## 8. Post-restore fixes

26. Edit the restored PVs. *(Likely: this fixed node affinity or host paths for the rebuilt node.)*
    ```sh
    kubectl get pv
    kubectl edit pv <pv-name>
    ```
    The two edited this time were:
    - `pvc-a660a4ac-019e-4708-a277-e8be382364c6`
    - `pvc-1b6ba2b6-049d-4fd7-b9be-b17e52c7a02e`
27. Edit the node. *(Likely: this re-added the labels or taints that the workloads expect.)*
    ```sh
    kubectl edit node server.hubble
    ```
28. Private registry: create `/etc/rancher/k3s/registries.yaml`, then:
    ```sh
    rc-service k3s restart
    cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml   # confirm it was picked up
    ```

## 9. Cleanup

- [ ] Delete `/tmp/k3s-config`. It was set to `chmod 777` and holds full cluster-admin credentials.
- [ ] Put `/etc/rancher/k3s/k3s.yaml` back to `600`. It was opened up to `644`, and fredo has his own copy now.
- [ ] Confirm that ufw and k3s are both in `rc-update` (default runlevel).
- [ ] Remove `server/db.bak` and `server/tls.bak` once the cluster is stable.

## Backup checklist for next time

Keep all of these on the NFS share (`/nfs/Hubble`), taken at the same moment:

- `etcd.tar.gz`: `/var/lib/rancher/k3s/server/db`
- `rancher-token`: `/var/lib/rancher/k3s/server/token`
- `server.tar.gz`: `/var/lib/rancher/k3s/server` (including `tls/` and `manifests/`)
- `k3s-pvcs.tar.gz`: PVC data
- `fstab`
- `/etc/rancher/k3s/registries.yaml`
- `/etc/network/interfaces`, `/etc/doas.d/20-wheel.conf` and `/boot/extlinux.conf`
