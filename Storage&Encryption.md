* **ZFS with Passphrase Encryption**:

  * Root and datasets encrypted at rest with a manual passphrase.
* **Automatic Unlock via USB**:

  * Generate a raw keyfile (`dd if=/dev/urandom bs=32 count=1 of=/root/zfs-key.bin`) and copy to USB stick.
  * Initramfs hook script (`/etc/initramfs-tools/scripts/local-top/zfs-unlock`) checks for the USB:

    ```bash
    if USB detected; then
      mount USB → zfs load-key -L file:///mnt/zfs-key.bin rpool/ROOT/ubuntu_xxx
    else
      zfs load-key -L prompt rpool/ROOT/ubuntu_xxx
    fi
    ```
  * Rebuild initramfs: `update-initramfs -u`.
  * Fallback to on‑screen passphrase if USB missing.