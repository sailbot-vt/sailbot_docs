---
title: CI/CD
description: CI/CD pipeline overview.
---

# <p style="text-align: center;"> Continuous Integration/Continuous Deployment (CI/CD) </p>

The purpose of the CI/CD pipeline is to automatically perform certain actions when we push changes to the main branch of the repository. Some examples of these actions include:

- Rebuilding binaries for both Intel and Arm CPUs
- Rebuilding the Docker image that the devcontainer uses and pushing that image to Docker hub
- Creating a new Github release which allows people to install Autoboat binaries via the `apt` package manager

All of the files related to the CI/CD pipeline can be found in the `.github` folder.  
If you want to to learn more about CI/CD with Github, please see the below links:
 
- <https://docs.github.com/en/actions>
- <https://docs.docker.com/build/ci/github-actions>

## <p style="text-align: center;">What Does the CI/CD Pipeline Do When We Do Various Actions on Github?</p>

| Feature | Pull Request (PR) | Push to `main` | Version Tag (`v*`) |
| :--- | :---: | :---: | :---: |
| **Verify Build** (amd64 / arm64) | ✅ | ✅ | ✅ |
| **Build & Download `.deb` Packages** | ✅ | ✅ | ✅ |
| **Push Images** (Docker Hub / GHCR) | ❌ | ✅ | ✅ |
| **Update Rolling Release** (`latest`) | ❌ | ✅ | ❌ |
| **Create Official Release** | ❌ | ❌ | ✅ |
| **Multi-Arch Manifests** | ❌ | ✅ | ✅ |

## <p style="text-align: center;">File Structure of the `.github` Folder</p>

<br>

### <p style="text-align: center;">`workflows` Folder</p>

The files in this folder are used to define the different workflows that should run when certain events happen. Currently it contains the following files:

- `build-and-release.yml`: This file is responsible for:
    - Building each devcontainer variant and pushing those images to Docker Hub
    - Building the Debian packages for both Intel and Arm CPUs and pushing those packages to our `apt` repository
    - Creating a new Github release when we push a new version tag (e.g. `v1.0.0`) to the repository

- `docker-bake.hcl`: This file is responsible for:
    - Defining the different Docker build targets that we want to build when we push changes to the main branch
    - Informing `build-and-release.yml` which Docker images to build and push to Docker Hub when we push changes to the main branch

- `build_ros_packages.sh`: A bash file which:
    - Builds and packs our ROS2 packages into Debian packages
    - Improves the readability of the `build-and-release.yml` file by separating out the logic for building/packing the ROS2 packages

To learn more about Github workflows, please see the following links:

- <https://docs.github.com/en/actions>
- <https://docs.docker.com/build/ci/github-actions>

<br>



#### <p style="text-align: center;">`debian_package_files` Folder</p>

This folder contains all of the files required to construct the Debian packages. This includes:

- Post installation commands (`postinst` files)
- Commands to run when removing the Debian packages (`prerm` and `postrm` files)
- Package requirements and descriptions (template files)

The `postinst`, `prerm`, and `postrm` files are subdivided by the individual Debian package since they each have different post installation and removal commands.

Some tutorials for building Debian packages:

- <https://linuxopsys.com/build-debian-packages>
- <https://blog.heckel.io/2015/10/18/how-to-create-debian-package-and-debian-repository>


## <p style="text-align: center;">What Are the Different Custom Debian Packages that We Build?</p>

!!!NOTE "Additional Information"
    To learn more about the microros agent see: <https://www.yahboom.net/public/upload/upload-html/1706694373/Install%20and%20start%20microros%20agent.html> (scroll all the way down for the example on serial communication).

- `autoboatvt-simulation`: This installs Gazebo and all of the custom ROS2 plugins required to run the simulation.

- `autoboatvt`: This is the base Debian package. It contains:
    - The autopilot code binaries, which are needed to run the autopilot
    - The jetson driver code binaries, which allow us to monitor the metrics provided by `jtop`

- `autoboatvt-microros-agent`: This package contains:
    - The `microros_agent` binaries, which allow ROS nodes to communicate with the firmware running on the Raspberry Pi Pico over serial

- `autoboatvt-firmware-dependencies`: This package contains:
    - Everything needed to develop firmware for the Raspberry Pi Pico, which includes:
        - `pico-sdk`, the official SDK provided by Raspberry Pi for developing firmware for the Raspberry Pi Pico
        - `picotool`, which allows us to flash programs and reboot the Raspberry Pi Pico through the terminal
        - `micro_ros_raspberrypi_pico_sdk`, which allows us to integrate microros with the Raspberry Pi Pico SDK

## <p style="text-align: center;">How Does Installing The Custom Debian Files Work?</p>

The main installation mechanism for the custom Debian packages is simply by placing the binaries or repository inside of `/opt/autoboat`. The `/opt` folder in Linux is generally reserved for installing additional *optional*, cough cough **opt**, software. For reference whenever you install ROS2 on your computer, it also installs to this folder. 

In order for programs on your computer to actually know where the Debian packages they require are installed to, they generally require you to run a "setup" bash script or set an environment variable pointing to the installation location in `/opt/autoboat`. Oftentimes this is done through modifying the `~/.bashrc` file, which is a bash script that is run every time you open a new terminal. This will automatically run the "setup" bash scripts and set any environment variables for the current terminal session so any program on your computer can seamlessly see the Debian packages you just installed.


**Installing a package** is a two-step process: first, you load the Debian package with the files you want the user to have, and second, you tell the package where those files should end up on their computer. Let's walk through two examples to make this concrete:

**Example 1 — Installing a ROS2 Package**

When you run `colcon build` inside a ROS2 workspace, it generates an `install` folder containing all of the compiled ROS2 nodes. To ship this as a Debian package, we:

1. Load the Debian package with that `install` folder
2. Tell the package to copy it to `/opt/autoboat/install` when the user runs `sudo apt install`
3. Use a `postinst` script to add the line `source /opt/autoboat/install/setup.bash` to the user's `~/.bashrc` file

That `source` command is what tells the user's ROS2 installation where to find all of our custom nodes and without it, ROS2 wouldn't know they exist.

**Example 2 — Installing a Simple Library (e.g. the Raspberry Pi Pico SDK)**

Some software doesn't need to be "built" in the ROS2 sense and it just needs to live somewhere on the computer and be pointed to. For the Pico SDK, we:

1. Load the Debian package with the pre-cloned SDK repository
2. Copy it to `/opt/autoboat/firmware_dependencies/pico-sdk`
3. Append `~/.bashrc` with a line that sets the `PICO_SDK` environment variable to that path so CMake can find it when compiling firmware

**Keeping `~/.bashrc` Clean and Manageable**

As we install more packages, each one needing its own additions to `~/.bashrc`, things can get messy — especially when it comes time to *remove* a package. If everything were written directly into `~/.bashrc`, we'd have to carefully find and delete the right lines without breaking anything else.

To avoid this, we only ever add a single line to `~/.bashrc`:

```sh
for f in /etc/profile.d/autoboatvt-*.sh; do [ -f "$f" ] && source "$f"; done
```

This line scans the `/etc/profile.d/` folder for any file starting with `autoboatvt-` and automatically runs each one on terminal startup. Instead of cluttering `~/.bashrc` directly, each Debian package gets its own dedicated setup script in that folder. For example:

- `autoboatvt-firmware-dependencies` creates `/etc/profile.d/autoboatvt-firmware-dependencies.sh` when installed
- When that package is removed via `sudo apt remove`, that script is automatically deleted
- On the next terminal startup, `~/.bashrc` simply doesn't find the file anymore, so nothing breaks

This keeps each package's setup logic self-contained and makes uninstalling clean and straightforward.
