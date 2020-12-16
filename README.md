## Idea

Scan to PDF automation... like thousands of other attempts out there.
But both convenient and rock solid, with no "hacking" of the Diskstation.

Most ideas, attempts and even tutorials out there end, where it gets interesting and hard.
This must end now ;)

## Purpose

I guess the most common thing out there is a scanner (with scan to network feature) a NAS, which takes the scanned documents and some devices (PC, Phone etc.) to consume the documents.

Some dream of automating everything, like sorting that PDFs automatically and stuff like that - I think that is bullshit and if you like, go on and improve some steps you find here (some might say even the automation of OCR is wasting of time ;)) 

I just want to get the PDFs from the scanner -> to the NAS -> process them with OCR -> place them somewhere so I can access them easily from my Mac. Nice to have would be to get something like an semi-duplex scan feature - as my scanner does not provide that feature.

## Hardware setup

* Synology Diskstation (new enough and enough power to run Docker)
* Brother MFC



Concrete requirements

- Scan documents to Diskstation as PDF
- Process the PDF automatically triggered by that new file
  - OCR
  - merge even (back) and odd (front) pages
- Put it in a shared Folder, which I can easily access with devices (ideally synced with Synology Drive)

And another one is to 

* do that in a low power consumtion way as possible!

Last one is a challenge, as watching for new files in a folder does require some power for the process itself in the cpu and the HDD access keeps the disks spinning all time - no hibernate (aka deep sleep).



## The Solution
### prepare
- setup a user *scanner*
- add a new shared folder "scans" (where the end result will be placed) with access for all real users and the user *scanner*
- add a second shared folder "scaninbox" and create a subfolder inbox in it with only access for the user scanner (optionally you can use the public shared folder and give scanner user access to a subfolder like /scans/inbox)

### scan
- setup your scanner to use that user to place the scanned PDFs on the shared scaninbox folder.

### OCR

[OCRmyPDF in a docker setup](https://ocrmypdf.readthedocs.io/en/latest/docker.html#installing-the-docker-image) is the way to go.

* Get it from the command line ([enable and use ssh as root on your DS](https://www.synology.com/de-de/knowledgebase/DSM/tutorial/General_Setup/How_to_login_to_DSM_with_root_permission_via_SSH_Telnet))

  ```
  docker pull jbarlow83/ocrmypdf
  ```

- configure and start an container to [watch your scan inbox folder](https://ocrmypdf.readthedocs.io/en/latest/batch.html?highlight=watcher#watched-folders-with-watcher-py) 

  - place the following content in a file docker-compose.yml

    ```
    version: "3.3"
    services:
      ocrmypdf:
        restart: always
        container_name: ocrmypdf
        image: jbarlow83/ocrmypdf
        volumes:
          - "/volume1/scaninbox/inbox:/input"
          - "/volume1/scans:/output"
        environment:
          - 'OCR_JSON_SETTINGS={"language": "eng+deu", "force_ocr": true}'
          - OCR_ON_SUCCESS_DELETE=1
          - OCR_OUTPUT_DIRECTORY_YEAR_MONT=0
        user: "<SET TO YOUR USER ID>:<SET TO YOUR GROUP ID>"
        entrypoint: python3
        command: watcher.py
    ```

  - replace `<SET TO YOUR USER ID>` and `<SET TO YOUR GROUP ID>` with the user and group id of your *scanner* user.

    * You can get that user id by `id -u scanner` resp the group id by `id -g scanner`
    * Do not forget the `:`between user id and group id ;)
    * modify the volume paths according to your setup
    * modify the OCR_JSON_SETTINGS to your need - [unfortunatelly the parameters are quite unobvious documented here](https://ocrmypdf.readthedocs.io/en/latest/api.html?highlight=OCR_JSON_SETTING#reference)

  - create and start the container
  
    ```
    docker-compose up -d
    ```

That's it! Every PDF that your scanner will send to the folder `/volume1/scaninbox/inbox` will be processed, PDF with the same name but OCRed will be placed in `/volume1/scan`  and the input PDF will be deleted fromt the inbox folder.

### Reduce Power consumption

As my Diskstation is 90% of the day in the idle state - I'd like to keep it that way. But the script in the Docker container is constantly checking the folder for new PDFs - that keeps the disks from hibernating.

Some might say, forget the power consumption of the disks, they will last longer by not hibernating at all. To those, step ofer the steps that follow ;)

It's hard to verify what prevents the DS from hibernating the disks, tough stuff.

Here are some links to that issue:

[official Synology knowledgebase](https://www.synology.com/en-us/knowledgebase/DSM/tutorial/Management/What_stops_my_Synology_NAS_from_entering_System_Hibernation)

[Unofficial checklist to test/ solve hibernation issues (quite outdated)](https://community.synology.com/enu/forum/17/post/11558?reply=53442)

[Enable Synology's hibernation debugger](https://shred.zone/cilla/page/446/enable-synologys-hibernation-debugger.html)

#### So here's the idea to avoid disk access while watching a folder

Using a ramdisk as the input folder. The script will run on a folder which is not a mount on the HDD. Instead it will run in the ram and thus should not touch the HDD.

- [create a triggered task in the Task Scheduler](https://www.synology.com/en-us/knowledgebase/DSM/help/DSM/AdminCenter/system_taskscheduler) to be started on  boot, run it as *root*

- place the following line in User-defined script

  ```
  mount -t tmpfs -o size=100m,mode=777 scanbox /volume1/scaninbox/inbox
  ```

  * Modify the `/volume1/scaninbox/inbox` to your inbox folder.
  * If you prefer more space for eventually bigger PDFs, increase the size of 100m to something bigger.

I can confirm, it works - the disks go to sleep after the idle time. And if some PDF triggers the watching Docker Image, the processing starts and the disk will spin up (if not already before). The result is placed in the scans output folder. If no further action occures, the disk will spin down after the idle time.

### Issues

- Do NOT try to place the input ram disk in a shared Synology Drive. It will break your setup (Synology warns about that, and also trying to share a Folder with some mounted drives will fail. But already shared folders will break if you place some mounts afterwards in it)
- The output can be a shared Synology Drive folder. But it seems that the writing from whithin the Docker image to that folder does not trigger the update. So the change does not trigger a sync to the clients. That's really sad, as it would be so neat to get the result pushed to the clients imediatelly after the processing is done.



## Where to go from here
1. automatic PDF merging of even and odd pages to emulate Duplex-ADF
   A solution might be something like a script based on the Docker image using pdftk

   https://registry.hub.docker.com/r/mnuessler/pdftk

   Even better would be some watching docker image, like the OCRmyPDF which just watches the input folder for files with the pattern front103.pdf and waiting for back192.pdf and merging those directly to the output and deleting the input.

2. Get output trigger change notification in a Synology Drive setup