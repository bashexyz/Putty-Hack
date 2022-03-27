# CSCM28 Coursework 2 (CVE-2021-33500)

## Students

- Bashiru Salami (2140326)
- Kin Ip Mong (2143876)

## Description

[PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/) is an open source tool for SSH and Telnet on different platforms. [CVE-2021-33500](https://nvd.nist.gov/vuln/detail/CVE-2021-33500) introduces a remote denial of service attack on Windows GUI using PuTTY version 0.74 or below. The attack is based on changing the title of PuTTY window rapidly, which creates a lot of [SetWindowTextW](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowtextw) calls, eventually causing Windows GUI becomes unresponsive. A proof of concept has been provided by [SSH-MITM](https://www.ssh-mitm.at/) and its [plugins](https://github.com/ssh-mitm/ssh-mitm-plugins). This exploit and its fixes can be demonstrated with the steps below. **Note that the demonstration should only be performed under controlled environment and consents from all parties concerned. All machines should be disconnected from the Internet when conducting the exploitation.**

## System Requirements

As this exploit is related to SSH connection and Windows GUI, at least two operating systems are required for this demonstration.

- SSH Server
    - Linux distribution is required
    - Root privileges are required
    - Python 3.6 or above is required
    - Debian-based Linux distributions are preferred

- SSH Client
    - Windows 7 or above is required
    - [PuTTY 0.74](https://the.earth.li/~sgtatham/putty/0.74/w64/putty.exe) (vulnerable version)
    - [PuTTY 0.75](https://the.earth.li/~sgtatham/putty/0.75/w64/putty.exe) (fixed version)

## Preparation

The following commands and steps are based on Kali Linux 2021.4 and Windows Home 10.0.19044. Minor adjustments may be required for different distributions of Linux and Windows, such as package manager, please refer to the documents of the installed operating system.

### SSH Server (Kali Linux 2021.4)

1. Download and install the Linux operating system

    - There is a list of Linux systems suitable for this demo, including [Kali Linux](https://www.kali.org/docs/installation/hard-disk-install/), [Ubuntu](https://ubuntu.com/tutorials/install-ubuntu-desktop#1-overview) and [Debian](https://www.debian.org/releases/stable/amd64/). The installation process of them may differ and it is the best to follow the official guides and documents. 

    - For Kali Linux 2021.4, the followings steps are taken to install the operating system, detailed guide please refer to [official guide](https://www.kali.org/docs/installation/hard-disk-install/)

        - Download the suitable installer image to an external drive from the [Download](https://www.kali.org/get-kali/#kali-bare-metal) page

        - Set the machine to boot from the external drive

        - Boot the machine and it should show the installer menu

        - Select the suitable build options

        - Wait until the installation is completed

2. Update the package lists

    ```bash
    sudo apt-get update
    ```

3. Install Python 3.6 or above (Python 3.9 is installed for this example) and `pip` (can be skipped if Python and `pip` is installed)

    ```bash
    sudo apt-get install python3.9 python3-pip
    ```

    - [Optional] Verify Python version

        ```bash
        python3 --version
        # Python 3.9.8
        ```

4. Install SSH-MITM version 0.6.3 (Plugins only compatible with SSH-MITM 0.6.3)

    ```bash
    python3 -m pip install ssh-mitm==0.6.3
    ```

    - [Optional] Verify SSH-MITM version

        ```bash
        ssh-mitm --version
        # SSH-MITM 0.6.3
        ```

5. Install SSH-MITM plugins

    ```bash
    pip install ssh-mitm[plugins]
    ```

    - [Optional] Verify SSH-MITM plugins version

        ```bash
        pip show ssh-mitm-plugins
        # Name: ssh-mitm-plugins
        # Version: 0.4
        ```

6. Install Uncomplicated Firewall(UFW) and enable it

    ```bash
    sudo apt-get install ufw
    sudo ufw enable
    ```

    - [Optional] Verify UFW status

        ```bash
        sudo ufw status
        # Status: active
        ```

7. Start SSH service

    ```bash
    sudo systemctl start ssh
    ```

    - [Optional] Verify SSH status

        ```bash
        sudo systemctl status ssh
        # ssh.service - ...
        #   Loaded: ...
        #   Active: active (running) since ...
        ```

8. Record the private IP address, which is `10.0.0.0` in this example, for SSH connection

    ```bash
    ifconfig
    # wlan0: ...
    #   inet 10.0.0.0 netmask ...
    ```

### SSH Client (Windows Home 10.0.19044)

1. Download [PuTTY 0.74](https://the.earth.li/~sgtatham/putty/0.74/w64/putty.exe)

## Exploitation

### SSH Server (Kali Linux 2021.4)

1. Start the SSH-MITM server

    ```bash
    ssh-mitm --ssh-interface puttydos
    ```

    - Sample output 

        ```bash
        -------------- SSH-MITM - ssh audits made simple --------------
        Version: 0.6.3
        Documentation: ...
        Issues: ...
        ---------------------------------------------------------------
        ...
                INFO listen interfaces 0.0.0.0 and :: on port 10022
        ```

2. Check and record the listening port number of SSH-MITM server (default listening port is 10022)

    ```bash
    listen interfaces 0.0.0.0 and :: on port 10022
    ```

3. Use UFW to allow SSH connection to the listening port

    ```bash
    ufw allow 10022
    ```

    - [Optional] Verify UFW status
        ```bash
        ufw status
        # Status: active
        #
        # To                         Action      From
        # --                         ------      ----
        # 10022                      ALLOW       Anywhere
        # 10022  (v6)                ALLOW       Anywhere (v6)
        ```

### SSH Client (Windows Home 10.0.19044)

1. Run the installed PuTTY 0.74

2. Connect PuTTY to the SSH server with the recorded IP and port number

    | Fields                    | Values                             |
    | ------------------------- | ---------------------------------- |
    | Host Name (or IP address) | 10.0.0.0 (the recorded IP address) |
    | Port                      | 10022 (the recorded port)          |
    | Connection Type           | SSH                                |

3. Attack is successful if the GUI hangs

## Fixes

According to [PuTTY 0.75 release note](https://www.chiark.greenend.org.uk/~sgtatham/putty/changes.html), a security fix regarding this exploit is published with this version. The fix deployed is adding a delay to the window modifications and hence prevent Windows API from being called rapidly.

### Fix before PuTTY 0.75

The naive approach to fix this vulnerability is by disabling the window title changing function. 

1. Open PuTTY configuration panel

2. In the `Category` panel on the left, select `Terminal`, then `Features`

3. Check the box of `Disable remote-controlled window title changing`

4. The PuTTY window title will not be changed by the SSH server and hence resolve the vulnerability

### Fix in and after PuTTY 0.75

In the updated version, the changing of the title is delayed. Although latency may exist in these versions, this weakness is no longer exploitable.

1. Download [PuTTY 0.75](https://the.earth.li/~sgtatham/putty/0.75/w64/putty.exe)

2. Run the installed PuTTY 0.75

3. Connect PuTTY to the SSH server with the recorded IP and port number

4. The same attack should fail

## Resources

- [Kali Linux](https://www.kali.org/)
- [Python Source Releases](https://www.python.org/downloads/source/)
- [SSH-MITM - ssh audits made simple - GitHub](https://github.com/ssh-mitm/ssh-mitm)
- [SSH-MITM Plugins - GitHub](https://github.com/ssh-mitm/ssh-mitm-plugins)
- [PuTTY 0.74](https://the.earth.li/~sgtatham/putty/0.74/w64/putty.exe)
- [PuTTY 0.75](https://the.earth.li/~sgtatham/putty/0.75/w64/putty.exe)


## References

- [PuTTY: a free SSH and Telnet client](https://www.chiark.greenend.org.uk/~sgtatham/putty/)
- [CVE-2021-33500 Detail - National Vulnerability Database](https://nvd.nist.gov/vuln/detail/CVE-2021-33500)
- [SetWindowTextW function (winuser.h)](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowtextw)
- [SSH-MITM - ssh audits made simple](https://www.ssh-mitm.at/)
- [SSH-MITM Docs - CVE-2021-33500](https://docs.ssh-mitm.at/CVE-2021-33500.html)
- [PuTTY release note](https://www.chiark.greenend.org.uk/~sgtatham/putty/changes.html)
- [PuTTY Commit on adding UPDATE_DELAY](https://git.tartarus.org/?p=simon/putty.git;a=commitdiff;h=d74308e90e3813af664f91ef8c9d1a0644aa9544)
- [UFW - Community Help Wiki](https://help.ubuntu.com/community/UFW)