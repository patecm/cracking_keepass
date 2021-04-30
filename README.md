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

## Installing HashCat
Some Linux instilations come with hashcat built in or a version you can download with apt-get. But, hashcat recommends compiling from their github, so that's what we're going to do. 
```
[user]$ git clone https://github.com/hashcat/hashcat.git
[user]$ cd hashcat
[user]$ make
[user]$ sudo make install
[user]$ cd ..
```
*Note: If you install it, cached kernels, session files, restore- and pot-files etc. will go to $HOME/.hashcat/*

Now check that hashcat is recognizing your devices, like your GPU  
```[user]$ hashcat -I```  

![Hashcat CUDA check](https://github.com/patecm/cracking_keepass/blob/f6e132d3ee7b745ccee8e4e6dfb4a2f46c56c80f/images/hashcat_cuda_info.png)

Note: If you choose to benchmark hashcat, you might get the warning:
```
Benchmarking uses hand-optimized kernel code by default.
You can use it in your cracking session by setting the -O option.
Note: Using optimized kernel code limits the maximum supported password length.
To disable the optimized kernel code in benchmark mode, use the -w option.

* Device #1: WARNING! Kernel exec timeout is not disabled.
             This may cause "CL_OUT_OF_RESOURCES" or related errors.
             To disable the timeout, see: https://hashcat.net/q/timeoutpatch
* Device #2: WARNING! Kernel exec timeout is not disabled.
             This may cause "CL_OUT_OF_RESOURCES" or related errors.
             To disable the timeout, see: https://hashcat.net/q/timeoutpatch
```
Per the [timeoutpatch documentation](https://hashcat.net/wiki/doku.php?id=timeout_patch) you need to make a config file:
```
[user]~$ cd /usr/share/X11/xorg.conf.d/ 
[user]~$ sudo vi 20-nvidia.conf
```
then add to the file '20-nvidia.conf'
```
Section "Device"
Identifier "MyGPU"
Driver "nvidia"
Option "Interactive" "0"
EndSection
```
If you're not familiar with vi or vim, hit 'esc' then :wq to save and quit the editor
You may need to reboot after this. You may not need to do it all though. Totally depends on how your system is set up.

## Get Cracking  
We're good to go!  
Now let's go back to our project directory and run the utility keepass2john
```
[user]~$ cd ~/Documents/crackpass
[user]~$ john/run/keepass2john CrackThis.kdb > CrackThis.txt
```
We can use that text file with JohnTheRipper. Unfortunately,  keepass2john starts the file with a prefix that hashcat doesn't recognize. If you open CrackThis.txt with a text editor, you'll see:  
>CrackThis.kdb:$keepass$*1*6000*0*cdee75c93f43...  
You could copy and past the very long hash into a new file without 'CrackThis.kdb':. Or you could let the system do it for you!  
```
[user]~$ john/run/keepass2john CrackThis.kdb | grep -o "$keepass$.*" >  CrackThis.hash
```
We know we have a keepass hash file, so let's find the settings in hashcat for that
```
[user]$ hashcat --help | grep -i "KeePass"
>  13400 | KeePass 1 (AES/Twofish) and KeePass 2 (AES)      | Password Managers
 ``` 
 If you look back through the hashcat settings (```hashcat --help```) you'll see the flags we need to run a dictionary attach on a keepass hash. 
* -m 13400 : Type of hash we are cracking (KeePass)
* -a 0 : Attack mode, 0=Dictionary Attack
* - w 2 : Optional workload profile 1=Low, 2=Economic (default)
* -d  (backend) or -D (OpenCL backend): Optional device type 1=CPU, 2=GPU
* -o CatCracked.txt : is the output file for the cracked passwords
* CrackThis.hash : input file with the hash value
* rockyou.txt : path to dictionary used for the attack (in current folder in this case)

Lets make sure we're in our project directory and run HashCat
```
[user]$ cd ~/Documents/crackpass
[user]$ hashcat -a 0 -m 13400 -o cracked_output.txt --outfile-format 2 CrackThis.hash rockyou.txt
```
![HashCat Results](https://github.com/patecm/cracking_keepass/blob/f6e132d3ee7b745ccee8e4e6dfb4a2f46c56c80f/images/hashcat_complete.png)
It was a pretty easy an unsecure password. 
Hashcat cracked it in about **2 seconds** on my machine!

cracked_out.txt contains the plain text password for you  

What if you want delete the output or accidentally overwrite an old password crack? No worries. Hashcat saves past cracks in the potfile at ```~/.hashcat/hashcat.potfile```  
By specifying the type of hash and original has file, you can print out the results again
```
hashcat -m 13400 --show -o cracked_output.txt --outfile-format 2 CrackThis.hash
```
And there you go. You're all set to get cracking!
