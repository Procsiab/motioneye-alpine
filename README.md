# Motion + MotionEye Alpine-based Container

[![Container Build](https://github.com/Procsiab/motioneye-alpine/actions/workflows/build-container-publish-dockerhub.yaml/badge.svg)](https://github.com/Procsiab/motioneye-alpine/actions/workflows/build-container-publish-dockerhub.yaml)

![Docker Image Version (latest by date)](https://img.shields.io/docker/v/procsiab/motioneye-alpine?label=Latest%20tag%20pushed%20on%20Docker%20Hub)

A Linux Container minimal distribution based on Alpine, which includes motion, the support for USB camera devices and also the MotionEye front end.

## Build the Container Image

Run the following command in this repo's directory:

```bash
podman build -f Containerfile.amd64 -t mycompany/motioneye-alpine
```

**NOTE**: You may want to use the `Containerfile.aarch64` to produce an image for an ARM64v8 target platform. In that case, the package `qemu-aarch64-static` must be installed.

## Run the Image

To launch in interactive mode and share the `/dev/video0` video device, run the following command (replace `podman` with `docker` if using the latter):

```bash
podman run --rm -it \
    -p 8765:8765 \
    --device=/dev/video0 \
    --name motioneye \
    docker.io/procsiab/motioneye-alpine:latest
```

**NOTE**: You may as well run MotionEye without attaching a video device to the container; in that case, omit the `--device` option from the command above

### Save media and configuration

To add persistence to the container, you may mount folders or volumes to the following paths inside the container:
- `/etc/motioneye`: the configuration folder; it must contain the default configuration file, that can be found [here](https://github.com/Motion-Project/motion/blob/master/data/motion-dist.conf.in)
- `/var/lib/motioneye`: the storage folder for all captured pictures and videos; it should be noted that MotionEye supports also FTP and SMB storage options through its configuration file ([learn more](https://github.com/motioneye-project/motioneye/wiki/Configuration-File))

### Permissions

TL;DR: to grant the necessary permissions to the MotionEye container there are two ways: a quick and diry one, and the correct one 😉 

#### Quick and dirty 💩 

Use `sudo` before the previous `podman run ...` command; also add the option `--privileged`. That's it.

#### The right way 😎 

Since you chose the right way, from now on I will assume that you are running either `podman` or `docker` in the so-called *rootless* mode; also, I am assuming that SELinux is enabled and in `Enforcing` mode.

1. enable the following SELinux booleans:
```bash
sudo setsebool -P container_use_devices 1
sudo setsebool -P container_manage_cgroup 1
```

2. create and install the following policy to allow the container to map the video device (just run the commands in order and you should be good to go):
```bash
cat << EOF > my-container-map-videodev.te
module my-container-map-videodev 1.0;

require {
	type v4l_device_t;
	type container_t;
	class chr_file map;
}

#============= container_t ==============
allow container_t v4l_device_t:chr_file map;
EOF
```

```bash
checkmodule -M -m -o my-container-map-videodev.mod my-container-map-videodev.te
semodule_package -o my-container-map-videodev.pp -m my-container-map-videodev.mod
sudo semodule -i my-container-map-videodev.pp
```

3. create a UDev rule to assign every `/dev/video` device to the group video (ensure that the group video exists):
```bash
echo 'KERNEL=="video[0-9]*",SUBSYSTEM=="video4linux",SUBSYSTEMS=="usb",ATTRS{idVendor}=="*",ATTRS{idProduct}=="*", GROUP="video"' | sudo tee /etc/udev/rules.d/15-webcam-video-group.rules
```

4. connect/turn on the video devices you want to share with the container, then apply the UDev rule (use your device's name in place of `/dev/video0`):
```bash
udevadm test /dev/video0
```

5. you can now run the container with the command shown in the previous section

## Troubleshooting

If you cannot see any device inside the MotionEye dialog, or even if you can add the video device to the interface only a grey background is displayed, I would suggest you to:

- If you are using SELinux in `Enforcing` mode, try to run the container after adding the `--security-opt label=disable` option to the `podman run ...` command; if that makes it work, than you still have some SELinux permissions to sort out
- Some webcams do not work well in Linux, others will show up as more than one `/dev/video` device: run the command `dmesg -w`, then try to unplug and then plug back in the device in question; then examine the Dmesg log to see what is happening
- Since MotionEye is running Motion behind the curtains, you can collect finer logs from running the `motion` command from inside the started container; you can change what device will be opened by `motion` by changing the default conf. file located at `/etc/motion/motion-dist.conf`
