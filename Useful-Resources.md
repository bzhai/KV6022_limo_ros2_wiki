* [The Linux command line for beginners](https://ubuntu.com/tutorials/command-line-for-beginners)
* [Python Cheat Sheet](https://github.com/ehmatthes/pcc_2e/blob/master/cheat_sheets/beginners_python_cheat_sheet_pcc.pdf)
* [ROS2 tutorial](https://docs.ros.org/en/humble/Tutorials.html)
* [ROS2 Cheat Sheet](https://www.theconstructsim.com/wp-content/uploads/2021/10/ROS2-Command-Cheat-Sheets-updated.pdf)

## Ubuntu on Windows using WSL2
Author: Jacob Swindell

Windows Subsystem for Linux (WSL) can also be used to run the simulator. If you are running an up-to-date Windows 10 or 11 install then you can [install WSL](https://cloudbytes.dev/snippets/how-to-install-wsl2-on-windows-1011) with:
```
wsl --install Ubuntu-22.04
```
For the smoothest experience with WSL, it is recommended to use Windows 11. This is because of the ability to display graphics [works out of the box on Windows 11](https://learn.microsoft.com/en-us/windows/wsl/tutorials/gui-apps). Windows 10 requires an [X server](https://sourceforge.net/projects/vcxsrv/) to be set up, running and connected to the WSL instance to display the simulator.

Once WSL has been installed then simply follow the instructions above for setting up the simulation natively, using the new WSL Ubuntu instance. If you run into issues with apt being very slow or not installing packages properly then you may have to [change your DNS in WSL](https://askubuntu.com/questions/1364984/dns-not-working-on-wsl).