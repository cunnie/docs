### Installing [Triton](https://triton-lang.org/main/getting-started/installation.html)

Create the Python **venv**:

```bash
mkdir ~/workspace/triton-mine
python3 -m venv ~/workspace/triton-mine/.venv
. ~/workspace/triton-mine/.venv/bin/activate
cd ~/workspace/triton-mine
echo .venv > .gitignore
git init
git add .gitignore
git ci -m"Ignore Python venv"
```

Let's install Triton:

```bash
pip install triton torch setuptools numpy matplotlib pandas ffmpeg
  Installing collected packages: filelock, triton
  Successfully installed filelock-3.15.4 triton-2.3.1
```

Now let's clone Triton from GitHub & install the tutorials:

```bash
git clone git@github.com:openai/triton.git ~/workspace/triton
cd ~/workspace/triton
pip install -e './python[tutorials]'
```

Switch gears back to "triton-mine":

```bash
cd ~/workspace/triton-mine
touch tut1.py
chmod +x tut1.py
nvim tut1.py
./tut1.py
```

Getting this error:

```
RuntimeError: Found no NVIDIA driver on your system.
```

```
sudo apt update && sudo apt upgrade
sudo add-apt-repository ppa:graphics-drivers/ppa
sudo apt update
lspci | grep -i nvidia
  03:00.0 3D controller: NVIDIA Corporation TU104GL [Tesla T4] (rev a1)
ubuntu-drivers devices
  nvidia-driver-545
sudo ubuntu-drivers install
```

And then we get this:

```
dpkg: dependency problems prevent configuration of nvidia-driver-545:
 nvidia-driver-545 depends on nvidia-dkms-545 (<= 545.29.06-1); however:
  Package nvidia-dkms-545 is not configured yet.
 nvidia-driver-545 depends on nvidia-dkms-545 (>= 545.29.06); however:
  Package nvidia-dkms-545 is not configured yet.
```

```bash
sudo nvim /usr/src/linux-headers-6.8.0-36/include/drm/drm_ioctl.h
```

```c
#define DRM_UNLOCKED 0
```

```
sudo shutdown -r now
```

<!--
When I run my small `triton.py` code from the [tutorial](), I get the following error:

```
ModuleNotFoundError: No module named 'triton.language'; 'triton' is not a package
```

I think this was caused by `triton` version 3.0.0, but when I `pip install
triton` after I removed it I got version 2.3.1.

-->
