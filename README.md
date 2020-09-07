# Raspberry-Pi-HQ-Camera-Livestream-with-Fedora

### Introduction:

The purpose of this instructional guide is so that you can use a Linux computer running Fedora with elgato cam link to stream video feed with extremely low latency from a Raspberry Pi using HQ camera in live meetings, such as Google Meet or WebEx or Skype or anything else. I've confirmed this works with Google Meet on firefox and chrome. 

Given the limitations of the Pi hardware the best we can hope for is 1080p, I have gotten a decently clear high quality picture with very close to 0 latency using this setup. I have seen other guides online which are in a similar realm of using elgato cam link with a external camera (not Pi camera) but to my knowledge I have not seen anyone else successfully be able to use the Raspberry Pi HQ camera with elgato cam link on Fedora, this has taken me significant debugging. My method uses ffmpeg to grab the video input from the elgato usb3 input, and outputs the image to a dummy device using v4l2loopback. 

The larger idea behind this project is building on capabilities of Fedora and Pi interaction Raspberry Pi HQ camera functionality with extremely low latency, and high amount of flexibility for command line utilization. There are other similar endeavours with Ubuntu used as the desktop OS which seem to have greater success with OBS studio for better graphical interface and less technical utilization of similar setup (elgato cam link with Pi). As I don't run Ubuntu I can't speak as to how well those work, just noting. 

If anyone has any suggestions for improvement please feel free to make a pull request / contact me. Would love to make this better.

Disclaimer:  I am not claiming that any of this is my original work, the only original work I've done is in bringing together information from diverse sources to achieve the desired outcome, as well as using some slightly different command line options. 

### Overview of hardware

For my setup I am using the following. I note the memory and processor because this requires a decent amount of processing power. I do not think this would work on Pi3, and it likely would not work with a low end processor on the desktop. 

- Raspberry Pi 4 with 4GB of memory, with the 6mm wide angle lens
- Fedora 32 desktop system with 4GB of memory, Ryzen 5 1600
- Elgato cam link 4k
- I used minihdmi (Pi4 hdmi size) to full size HDMI converter to plug into the Pi, then a 6foot HDMI cord from that to the elgato cam link. The elgato cam link plugs into a usb3 port on the Fedora system. Must be usb3. 

### Fedora Setup

Packages setup: 

- you will need v4l2loopback and b4l2loopback-dkms, I got this from the Fedora Copr repository here: [https://copr.fedorainfracloud.org/coprs/sentry/v4l2loopback/](https://copr.fedorainfracloud.org/coprs/sentry/v4l2loopback/)
- Note: many guides online say to build v4l2loopback from source, I would suggest you do not do that and just use  the above. For fedora, I was able to build v4l2loopback from source easily however v4l2loopback-dkms seemed a bit tricky. I think I may have missed that part. I noted that in Ubuntu guides for this type of setup, v4l2loopback-dkms is always used, so I suspect that without the dkms part you will likely hit issues.
- You will also need ffmpeg.
- there is a colorspace issue on elgato with Linux (it isn't supported on Linux hardware, technically, but it works!) and in order to workaround this I used this repo [https://github.com/xkahn/camlink](https://github.com/xkahn/camlink) - you will need to clone the repo and run with "make"

Initializing for streaming:

- with v4l2loopback package installed, you can now run this command to intialize the module. The exclusive_caps option is critical so that the dummy video interface is exposed to Google Chrome. You may have success on other browsers without that option. If for any reason you run the modprobe command without those options, you will need to unload that module and reload it.

~~~
sudo modprobe v4l2loopback devices=1 exclusive_caps=1
~~~
### Raspberry Pi Setup

On the Raspberry Pi, the first thing you need to do is ensure that you have a properly functioning camera with video output that shows full color and a decent framerate. The best way to test this is to connect the Pi to a monitor with HDMI cord, no cam link. Then do a simple test of raspivid, such as:

~~~
raspivid -t 0
~~~

If that doesn't work as expected, work through the official Raspberry Pi documentation for the HQ camera, then continue once you see the output you expect. 

Due to the considerations to 1. you may have to reboot a number of times on the Pi and this will speed up shutdown and reboot and 2. you likely want to lower the overall cpu and memory load on the Pi, I would suggest you do not boot into the graphical desktop on the Pi, but instead only boot to terminal. You can do this in the "sudo raspi-config" menu. 

In the /boot/config.txt file, place the following lines, what this does is set the output rate to HDMI_CEA_1080p60 (which equals mode 16). Without this, the Pi does a handshake with the elgato, and as it doesn't support linux, it tells the Pi to use the incorrect output rate. To my undertstanding this forces 1920x1080p at 60ghz. 

~~~
hdmi_drive=2
hdmi_group=1
hdmi_mode=16
~~~

reference:  [https://www.raspberrypi.org/forums/viewtopic.php?t=5851](https://www.raspberrypi.org/forums/viewtopic.php?t=5851)

I also suggest that you increase the gpu memory. I placed this line in the /boot/config.txt file to set 384mb of memory to the gpu.

~~~
gpu_mem=384
~~~

This last command should help you stop the HDMI from blanking due to inactivity, if you run into that issue try using this in /boot/config.txt

~~~
hdmi_blanking=1
~~~

### Streaming

After your Raspberry Pi and Fedora systems are booted, and you are connected via the elgato cam link 4k, then you should have a monitor with the Fedora system terminal open, as well as another terminal connected with ssh to the Pi 4 (over wifi or ethernet). 

From the **Fedora** **terminal**, run your ffmpeg command like so. Note that this is a result of a lot of taking commands and options from random sources. It works for me. You could probably make it better.

~~~
LD_PRELOAD=/home/admin/Documents/av_util/camlink/camlink.so \
ffmpeg -f v4l2 -input_format yuyv422 -vcodec rawvideo \
-framerate 30 -video_size 1920x1080 -i /dev/video0 -pix_fmt yuv420p \
-c:v libx264  -preset ultrafast -codec copy -f v4l2 /dev/video2 -loglevel debug
~~~

[some notes, I am not an ffmpeg expert, read the docs for authoritative information]
- LD_PRELOAD command part is from the Repo where we got the workaround to disable 2 out of 3 of the yuv outputs. This helps ensure that there won't be color issues and that the ffmpeg application uses the correct output stream from the elgato output. You will need to change the file path to match where you downloaded this git repo.
- -f vl42 uses the v4l2loopback driver, so you can use the /dev/video0 as an input
- input_format yuyv422 sets the input format of the video
- vcodec rawvideo sets the codec as raw video, not sure if this part is necessary or helpful
- framerate 30 should be matching to what you use on the Pi.
- video size should also match the options from the Pi
- input /dev/video0 points to the video stream from the elgato device and uses it as an input
- pix_fmt yuv420p sets the output format
- -c:v libx264 sets the libx264 decoder from the vlc project, I added this myself and I think it helps reduce load and latency.
- preset ultrafast is also designed to help reduce latency
- codec copy -f v4l2 sets to copy the decoded libx264 output to the v4l2loopback dummy device output
- I reccomend using loglevel debug always for better visibility into what ffmpeg is actually doing. But it is a lot of output.

### warning:

with my setup, the above syntax will work exactly ONCE after each time the elgato cam link is plugged in. If it's working you should see frame rate data at the bottom of the output. If its not working it will show some output and hang. If you want to stop the stream with Ctrl+C and then re-start the stream, you need to completely unplug the elgato cam link from your usb3 (MUST be usb3) port and then plug it back in, and try it again. I think this is due to the ffmpeg process somehow locking up the device, then the stream is interrupted in an unpleasant way, the entire device must be re-initialized. This is a bit of a nasty workaround so if anyone can help here, please do.

From the **Raspberry Pi** terminal, first I'd just check that your system is properly booted and is running. Run any command, such as "top" to look at your cpu and memory load, before you run any more commands.

At this point, I'd go back to your Fedora system, and open a Google Meeting and you should see the dummy interface initialize and you should see the terminal of the Pi. If you don't get any video feed, or elgato feed, try clicking settings in the bottom right and manually selecting "dummy interface" on the video output.

Once you're able to verify that you see the Pi terminal, you can now run raspivid command to start your stream. I'm still testing different commands using raspivid and raspidyuv. Here are some examples of syntax you can try:

~~~
raspividyuv -t 0 --width 1920 --height 1080 -fps 30
~~~
or
~~~
raspivid -t 0 -w 1920 -h 1080 -fps 30 -b 120000
~~~

Note that while your first tempation may be to increase all the settings to their maximums, try to keep an eye on "top" output on the Pi and on your Fedora system. If you use high quality settings, you may see your processor hit 120-200% utilization, which probably isn't good. Using variations on the first command, raspividyuv, I was able to get consistent output using about 50% cpu on the Fedora system. 

That's it! Just remember, if you end the ffmpeg stream and want to start it again, you need to manually unplug and re-plug in the elgato cam link on your Fedora system. Otherwise the command will just hang. 

### Debugging / Helper utilities [addendum]

- this is a helper command that will allow you to query the available video devices on your Fedora system using v4l2loopback - this will help you make sure your ffmpeg syntax is correct, that you are specifying the correct device, such as /dev/video0 for example. You want the first /dev/ device to be the input (from elgato/Pi) and the second /dev/ to be the dummy video output.

~~~
v4l2-ctl --list-devices
~~~

- if you run into issues with being able to show your dummy video interface in Google Chrome, you may want to run this command on Fedora that will help you check if you properly have exclusive_caps option enabled. If its not enabled, you need to unload and reload the module with correct options. If you hit issues there, just reboot and try again.

~~~
cat /proc/modules | cut -f 1 -d " " | while read module; do  \
echo "Module: $module";  if [ -d "/sys/module/$module/parameters" ]; \
then   ls /sys/module/$module/parameters/ | while read parameter; \
do    echo -n "Parameter: $parameter --> ";    \
cat /sys/module/$module/parameters/$parameter;  \
 done;  fi;  echo; done | egrep -i v4l2 -C 5
 ~~~

- this is a helper command on the Raspberry Pi to help you check what type of output you're sending. You want to be sending 1920x1080p at 60ghz, that's what worked for me. I would only suggest this after you've done some more basic checks.

You can dump the EDID of the monitor with 

~~~
# tvservice -d file.txt
~~~

and then parse the edid with

~~~
edidparser file.txt
~~~

Enjoy
