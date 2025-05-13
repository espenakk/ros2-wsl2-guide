# A guide to configuring WSL2 for using ROS2 on Windows 11

## Install Windows Terminal

First of all I recommend installing Windows Terminal if you haven't got it already.

[Install Windows Terminal](https://aka.ms/terminal)

## Install WSL2

Open PowerShell in **administrator** mode by right-clicking and selecting `Run as administrator`, enter the following command, wait for the install to finish and then restart your machine.

```powershell
wsl --install
```

WSL will install the latest long term support version of Ubuntu by default, this is exactly what we want for our ROS2 dev environment. If it for some reason fails to install Ubuntu then you can install it manually with this command:

```powershell
wsl --install -d Ubuntu
```

## Post install

Open Windows Terminal and click the arrow on the new tab icon and open Ubuntu. You will be prompted to choose a UNIX username and password.

> ⚠️ You will not see that you are typing in the terminal when inputting your password. This is normal.

When you have made a user you will be moved to your Ubuntu home directory. This is indicated in the terminal by `your-username@your-wsl-machine-name:~`, where the tilde character `~` is short for the path to your home folder.

The next step is to update the Ubuntu installation. You can do that by running the following command in the Ubuntu terminal.

```bash
sudo apt update && sudo apt upgrade -y
```

## Enable GUI Applications

To run Linux GUI apps, you should make sure you have the latest GPU drivers matching your system. This is needed in order to enable you to use a virtual GPU (vGPU) so you can benefit from hardware-accelerated OpenGL rendering.

- [**Intel** GPU driver](https://www.intel.com/content/www/us/en/download/19344/intel-graphics-windows-dch-drivers.html)
- [**AMD** GPU driver](https://www.amd.com/en/support)
- [**NVIDIA** GPU driver](https://www.nvidia.com/Download/index.aspx?lang=en-us)

## WSL2 Networking

To enable networking that reliably works you need Windows 11 Pro. This is because you need the Hyper-V tools in order to create a virtual network switch. For Windows 11 Home you can skip this section and use the less reliable workaround presented later.

### Hyper-V

Let’s check if Hyper-V is enabled. Do that by running this command in an administrator PowerShell prompt:

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

If it is disabled, run the following command to enable Hyper-V. After it is done installing, restart your computer.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

### Creating the Virtual Network Adapter

After you have Hyper-V installed, we can move on to creating the virtual network adapter. Start by running this command (in an administrator PowerShell) to get a list of your physical adapters:

```powershell
Get-NetAdapter
```

You need to choose either an Ethernet adapter or WiFi adapter to link the virtual switch to. To create the switch, run the following command. If you are using an Ethernet adapter, or your WiFi adapter has another name, replace `"WiFi"` with the name you got from the previous command:

```powershell
New-VMSwitch -Name "External Switch" -NetAdapterName "WiFi" -AllowManagementOS $true
```

## Configuring WSL2

Now we will configure WSL2 to use the virtual network adapter we just created. We will also configure the amount of RAM and swap space we want to make available to WSL2. We will do this by entering this command in PowerShell. It will open the configuration file in Notepad:

```powershell
notepad .wslconfig
```

If you have 32GB of RAM (or more) you can just replace your config file with this:

```ini
[wsl2]
swap=8589934592
memory=25769803776
networkingMode=bridged
vmSwitch="External Switch"
```

If you have less than 32GB, then you need to replace the memory amount with:

- `memory=12884901888` for 16GB (or more)
- `memory=6442450944` for 8GB (or more)

> ⚠️ You can allocate less, but you will likely encounter issues with building large ROS2 packages.

If you have allocated the suggested amount and still encounter memory-related issues when building ROS2 packages with `colcon`, you can try increasing the swap amount to:

```ini
swap=17179869184
```

## Workaround for Windows 11 Home

Since we can’t make a virtual network adapter without Hyper-V, we can do the next best thing: mirror the Windows network adapter. This will enable the WSL2 installation to communicate with other devices on your LAN.

This is built into WSL2, so the only thing we need to do is enable it in the config file:

```powershell
notepad .wslconfig
```

If you followed the previous step, you should already have stuff in your config file. Remove:

```ini
vmSwitch="External Switch"
```

Replace:

```ini
networkingMode=bridged
```

with:

```ini
networkingMode=mirrored
```

Your config should now look like this:

```ini
[wsl2]
swap=8589934592
memory=25769803776
networkingMode=mirrored
```

Next step is to allow inbound connections. This is needed in order to be able to detect ROS2 nodes running on other machines. Do it by running this command in an administrator PowerShell:

```powershell
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
```

## Some "Nice to Haves"

If you want to open a folder from WSL in windows explorer, you can do so by typing `explorer .`

If you want to seamlessly open your Ubuntu projects in VSCode, follow this tutorial to set up VSCode for WSL [Get started using VS Code with WSL | Microsoft Learn](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode)

## Install ROS2

Now WSL is up and running, the final step is to install ROS2 and start testing that everything works. Follow the guide on how to install for Ubuntu (deb packages) from the ROS2 documentation. [Ubuntu (deb packages) — ROS 2 Documentation: Jazzy documentation](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)

## Troubleshooting

If you get issues with the `ros2 node list` command, it could be that the daemon service is not running. This command should fix it:

```bash
ros2 daemon start
```

If you still have issues detecting nodes on other machines, it is likely that the Windows firewall is blocking some of the traffic. Run these commands (in an administrator PowerShell) to make exceptions for ROS2 ports, and afterwards reboot your machine:

```powershell
New-NetFirewallRule -DisplayName "ROS2 UDP 7400-7600" -Direction Inbound -Action Allow -Protocol UDP -LocalPort 7400-7600
```

```powershell
New-NetFirewallRule -DisplayName "ROS2 UDP 7400-7600 Outbound" -Direction Outbound -Action Allow -Protocol UDP -LocalPort 7400-7600
```

## References

[Install WSL](https://learn.microsoft.com/en-us/windows/wsl/install)

[Accessing network applications with WSL](https://learn.microsoft.com/en-us/windows/wsl/networking)

[Set up a WSL development environment](https://learn.microsoft.com/en-us/windows/wsl/setup/environment#set-up-your-linux-username-and-password)

[Get started using VS Code with WSL](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-vscode)