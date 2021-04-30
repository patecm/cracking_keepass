# Cracking a Keepass database
A tutorial on using JohnTheRipper and Hashcat to crack keepass hash files when you've lost the password    


*DISCLAIMER: This tutorial was created to help me retrieve a KeePass password I lost and utilize my GPU for cracking the hash. It's not to be used for malicious purposes. Please only crack your own lost passwords. A portion of this tutorial is based on an older version at [RubyDevices](https://www.rubydevices.com.au/blog/how-to-hack-keepass) It's been updated for newer operating system and library releases.*   

## Setup
First we need to start with a sample database. You might be anxious to get started on your current, personal database right away, but using a sample with a known password is a good way to get familiar with the process and doublecheck what you're doing is correct. It's also easier to troubleshoot any problems if you already know the password and can work backwards. A sample keepass database, called CrackThis.kdb, can be found in the repository here
Either download from the link above and move into your project directory (called crackpass in my case)   
```
 mkdir ~/Documents/crackpass
 cp ~/Downloads/CrackThis.kdb ~/Documents/crackpass/
 ```
 or download it directly from github  
 ```
  mkdir ~/Documents/crackpass
 cd ~/Documents/crackpass
 wget https://github.com/patecm/cracking_keepass/raw/main/CrackThis.kdb
 ```
 We also need a dictionary for attacking the hash file. There's been lots of password dumps throughout the years. Feel free to Google around and find your own if you'd like. Some version of Linux, like Kali, have wordlists built in already. The most popular from the Kali distribution is called 'rockyou' and is based on the [2009 databreach](https://en.wikipedia.org/wiki/RockYou) of a company by the same name. 
Since I'm running Ubunut 20.04 and not Kali, I had to download the dictionary. In the crackpass directory run:  
```
wget https://gitlab.com/kalilinux/packages/wordlists/-/raw/kali/master/rockyou.txt.gz
gzip -d rockyou.txt.gz
```
Right now, I'm running on Ubuntu 20.04 with a NVIDIA GTX1080Ti with 32GB 2400GHz RAM and an i9-9900k processor. Third party, including proprietary drivers, were installed when I initially set up the dual-boot Ubunut/Windows machine. You can check if the NVIDIA drivers are installed by going to *Software & Updates > Additional Drivers*   
![Nvidia Drivers](https://github.com/patecm/cracking_keepass/blob/8e1a69810ebb59867a63edc2c9ea481797b38dbe/images/nvidia-drivers.png)  

You can check you NVIDIA and CUDA installs with:
```
nvcc --version
nvidia-smi
```
If you have the NVIDIA driver installed, but not the CUDA tool kit, install that with
```
sudo apt install nvidia-cuda-toolkit
```

If everythin is good, you should see something like this
![nvidia-smi](https://github.com/patecm/cracking_keepass/blob/8e1a69810ebb59867a63edc2c9ea481797b38dbe/images/nvidia-smi.png)

You also need OpenCL, even with CUDA installed. Get the version for NVIDIA
```
sudo apt-get -y install nvidia-opencl-dev opencl-headers
```
Lastly, check your OpenSSL installation. As of writing, the most current version of [OpenSSL](https://www.openssl.org/) is 1.1.1h. You can check your version in terminal

```
openssl version
OpenSSL 1.1.1h  22 Sep 2020
```
If it's not installed on your system, I recommend following the instructions [here](https://cloudwafer.com/blog/installing-openssl-on-ubuntu-16-04-18-04/) or [here](https://stackoverflow.com/questions/3016956/how-do-i-install-the-openssl-libraries-on-ubuntu) but download the most up-to-date version instead of the one mentioned on those pages.

## Installing John the Ripper  
There's two tools we're going to use: John the Ripper and Hashcat. JtR will process our keepass database file and extract the hash for it. 
The original version ([found here](https://github.com/piyushcse29/john-the-ripper)) hasn't been updated in years and no longer supports OpenSSL versions > 1.1.0 Fortunately, our friends as Openwall have [kept it updated](https://www.openwall.com/john/).   
I'm going to install the package locally in a folder for my keepass projects, called but you can install it wherever you like. Just update the path for your system in the code below.  The JtR documentation recommends installing several other packages, so we'll do that now too.
```
[user]~$ sudo apt-get -y install build-essential libssl-dev git zlib1g-dev
[user]~$ sudo apt-get -y install yasm libgmp-dev libpcap-dev pkg-config libbz2-dev
```
Next, clone the latest version of JtR from the GIT repo and build it
```
[user]~$ cd ~/Documents/crackpass
[user]~$ git clone git://github.com/magnumripper/JohnTheRipper -b bleeding-jumbo john 
[user]~$ cd  john/src
[user]~$ ./configure && make -s clean && make -sj4
```
Install tab completion
```
[user]~$ sudo make shell-completion
```
Now test your build. This took about 5 minutes on my machines
```
[user]~$ cd ~/Documents/crackpass/john/run
[user]~$ ./john --test=0
```
![JtR test](https://github.com/patecm/cracking_keepass/blob/63aa8d3983500791edf2e0f56926566811f6fdff/images/john_test.png) 
And (optionally) benchmark the build. **Note that this can take 25+ minutes and is NOT required**
```
[user]~$ ./john --test
```

