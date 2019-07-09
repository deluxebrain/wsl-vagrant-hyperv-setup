# wsl-vagrant-hyperv-setup

How to use Vagrant with Hyper-V within WSL *with some very important caveats*.

Note that the caveats seriously limit the capabilities of this setup as a development environment.
I would only recommend this approach as a means to launch a "readonly" environment where (for example) you do not need to update files 

## TL;DR

1. Enable Hyper-V and set the `VAGRANT_DEFAULT_PROVIDER` environment variable to "hyperv"
2. Install Vagrant from the official Vagrant repository
3. Set the `VAGRANT_WSL_ENABLE_WINDOWS_ACCESS` environment variable to "1"
4. Store your Vagrant project repos within your user home directory on the Windows file system
5. Set the `VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH` environment variable to your Windows user home folder
6. Run WSL as administrator
7. Select the "Default Switch" Hyper-V switch

## Important caveats

1. Hyper-V networking cannot be configured via Vagrant

    Vagrant cannot setup networking rules on the Hyper-V swith.
    For example, the following is commonly used with VirtualBox but *will not work* with Hyper-V:

    ```sh
    # Configure 127.0.0.0:5000 to NAT across to the guest:50
    config.vm.network :forwarded_port,
        guest: "5000",
        host: "5000"
        auto_correct: false
    ```

    Therefore if you want to use Vagrant and Hyper-V with NAT translation you wll need to set the associated rules up for yourself.
    Unfortunately, the Hyper-V manager does not expose functionality to perform this step via the UI and hence you will need to drop down to PowerShell.

    One workaround is just to the use the IP address of the guest directory - i.e. not go via NAT.

2. Two-way synced folders are not supported

    Vagrant support for two-way synced folders is via either VirtualBox or SMB.
    The former is obviously not applicable, and unfortunately the latter does not work under WSL.

    Any synced folders setup within Vagrant will therefore fall back to rysnc and be one-way with sync performed at up and reload.

## Project dependencies

- vagrant ( installed from official vagrant repository )
- direnv ( optional )

    Used to automatically source the `config.env` file when entering the project directory.

    ```sh
    # from WSL home
    vim ~/.profile

    # append the following:
    eval "$(direnv hook bash)"
    ```

    *reload shell*

## Detailed setup notes

1. Enable Hyper-V

    Vagrant on WSL can work with both VirtualBox and Hyper-V.
    As Docker for Desktop requires Hyper-V, consolidate on Hyper-V and enable it as follows:

    ```sh
    # from PowerShell ( administrator )
    Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
    ```

    *reboot machine*

    ```sh
    # from WSL home
    vim ~/.profile

    # append the following:
    export VAGRANT_DEFAULT_PROVIDER="hyperv"
    ```

2. Installing Vagrant

    Vagrant must be installed within the Linux distribution used with WSL and from the official Vagrant repository.

    ```sh
    # https://www.vagrantup.com/downloads.html
    vagrant_url="https://releases.hashicorp.com/vagrant/2.2.5/vagrant_2.2.5_x86_64.deb"
    vagrant_filename="$(basename "$vagrant_url")"
    wget "$vagrant_url"
    sudo dpkg -i "$vagrant_filename"
    rm -f "$vagrant_filename"
    vagrant --version
    ```

3. Enable WSL to access Windows features

    See <https://www.vagrantup.com/docs/other/wsl.html#windows-access>

    ```sh
    vim ~/.profile

    # append the following:
    export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
    ```

4. Using synced folders

    See <https://www.vagrantup.com/docs/other/wsl.html#synced-folders>

    Support for synced folders for Vagrant running within WSL is limited to the `DrvFs` file system.
    One approach is therefore to store all Vagrant project repositories on the Windows file system ( under `/mnt/c` ).
    A convenience symlink can then be use to reference this from the WSL `VoIFs` file system.

    ```sh
    ln -s /mnt/c/Users/<username>/source ~/source
    ```

5. Linux file permissions

    See <https://www.vagrantup.com/docs/other/wsl.html#windows-access-1>

    Vagrant will not apply Linux file permissions to files shared from the Windows file system.
    This can cause certain permission checks to fail, such as thought associated with `vagrant ssh`.

    Vagrant will ignore these file permission checks for projects within the path defined by the `VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH` environment variable.

    One approach is therefore to set this environment variable to point to your user home folder within the Windows file system.

    ```sh
    vim ~/.profile

    # append the following:
    # tell vagrant to ignore file permissions for files in the following path:
    export VAGRANT_WSL_WINDOWS_ACCESS_USER_HOME_PATH="/mnt/c/Users/<username>"
    ```

6. Running vagrant within WSL

    The Hyper-V provider requires that vagrant is run with administrative priviledges.
    As such, the WSL console should be launched using `run as administrator`.

7. Using Hyper-V networking with vagrant

    Vagrant cannot natively control Hyper-V networking, and hence any networking setup needs to be performed manually against Hyper-V before launching vagrant. If multiple Hyper-V switches exist on the system, vagrant will prompt for confirmation of the switch to use at time of vagrant up as follows:

    ```sh
    Please choose a switch to attach to your Hyper-V instance

    1) DockerNAT
    2) Default Switch
    ```

    The "Default Switch" that is configured as part of Hyper-V is sufficient for most usecases. This is an internal switch that supports NAT for Internet traffic egress.

    Note that support for inbound NAT rules on the switch is not exposed via the Hyper-V manager and if required will need to be setup using PowerShell. The sensible alternative is to just use the guest IP address instead and forget about trying to NAT to the guest via the loopback address.

## Vagrantfile authoring

1. Vagrant box selection

    The vagrant box referenced in your Vagrantfile must support Hyper-V.
    Use the public box catalog to find one as appropriate.

    See <https://app.vagrantup.com/boxes/search?order=desc&page=1&provider=hyperv&q=ubuntu&sort=updated&utf8=%E2%9C%93>

    Generally speaking, and for the case of Ubuntu, this will mean using the Bento distribution as there is no official Ubuntu distribution that supports Hyper-V.
