# Installing Vivado on Ubuntu

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

If the installation hangs on generating installed devices, try the following steps (including a reboot) before re-trying the installation:

```
sudo apt-get install -y python3-pip
```

```
sudo apt-get install -y libstdc++6
```

```
sudo apt-get install -y libgtk2.0-0
```

```
sudo apt-get install -y dpkg-dev
```

```
sudo apt-get install -y libtinfo5 libncurses5
```

If the previous command did not work, try this:

```
sudo apt update
wget http://security.ubuntu.com/ubuntu/pool/universe/n/ncurses/libtinfo5_6.3-2ubuntu0.1_amd64.deb
sudo apt install ./libtinfo5_6.3-2ubuntu0.1_amd64.deb
```

Reboot the VM or your PC and try the installation agian.

(Without ibtinfo5 the application will not start and without libncurses5 the simulation will fail)
