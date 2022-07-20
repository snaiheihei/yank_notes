# seabee script

- seabee script Source code
::: tip seabee script
source code 2022-06-23
:::


```shell
#!/bin/bash
#
# Copyright 2020 General Electric Company.  All rights reserved.
#


BNAME=$(basename $0)

GRUBINSTALL=grub2-install
GRUB_FILE=$DATAMNT/boot/grub2/grub.cfg

SYNC_WITH_PROG="rsync -avh --delete --progress --size-only --copy-links "
SAFE_UNMOUNT_FOLDER="umount "
SAFE_UNMOUNT_DEVICE="udisksctl unmount --force -b "
SAFE_REMOVE="eject "

##########################
# OS CONTENT SETUP
USBISO_LOC=""
OS_CONTENT=""
##########################


##########################
BIOS_LABEL=WXBIOS
EFI_LABEL=WXEFI
EFI_FILE_LABEL=WXEFIFILE
DATA_LABEL=WXDATA

DEVTOUSE=""
DATADEV=""
EFIDEV=""
LOOPDEV=""
USBDEV=""
BIOSDEV=""
BOOTMODE="A"

TMPSTORE="/tmp/$BNAME.$$"
LOGFILE=$(readlink -f .)"/$BNAME.$$.trace"

EFIMNT="${TMPSTORE}/efi"
DATAMNT="${TMPSTORE}/data"

TMPMOUNT="${TMPSTORE}/tmp"
LOOPBACK="${TMPSTORE}/loopback.img"
USBISO_MNT="${TMPSTORE}/usbiso"
OSVERSIONLST=(6 7)
DISCLIST=(MrpApps MrTest MrpResSrv Brainwave)
##########################
FORCE=0
MRTF=0
ISPRODUCTION=0
MODE='?'
INTEGRITY_CHECKS=1

FLOOPBACK=""
LOOPBACK_SIZE=16
BUILD_NAME=""
OS_LABEL="7"
VREOS_LABEL="7"
APPS_LOC="/b/pvr2"
HL6_CONFIG_LOC=""
HL7_CONFIG_LOC=""
MRTFENABLE="765890"
GRUB_FILE=""
LOOPLOCK="/tmp/looplock.tmp"
SAVEINFOFOLDER=""
MYLOOPLOCK=0
DONT_EJECT=0
SKIP_GRUB_INSTALL=0
IGNORE_UNMOUNT_ERRORS=0
GRUB_TIMEOUT=30
# Vega wants to ship with robust mode available - set this back to (E)
# when robust is no longer the default
#ALLOWROBUST=(E)
ALLOWROBUST=(0 E)
##########################


function usage
{
    cat <<EOF
Usage: read the code
    -b | --buildname        : Name of build to load (e.g. MR27.0_EA_xxxx)
    -a | --apps             : Location of the application build folder (for testing/debugging purposes only)
    -h | --osversion        : HELiOS major version to use (e.g. 6, 7 or A [for all]) default = $OS_LABEL
    -o | --oslocation       : The location where the OS content can be found (default is taken from the MrUSBInstaller.iso with each build)
    -f | --force            : Not implemented
    -i | --integrity        : Changes the USB integrity checks. Prohibited if this is a production stick. 0 = none, 1 (default) = all
    -l | --loopback         : Name of the file to make into a virtual device and write to
    -s | --loopsize         : Size of the loopback virtual device to use (default $LOOPBACK_SIZE GB)
    -S | --Sinfo            : Source directory for saveINFO (Basename should be saveINFO)
    -d | --usbdev           : Raw OS device that represents the USB stick to use
    -t | --mrtf             : Will make the installation start automatically once the USB is booted
    -z | --vbox             : Installer will be used for VirtualBox creation. Implies --VM.
    -V | --VM               : Install as show system and update xorg for VM. For VBox requires manual install of guest addition and resizing view to 1920x1200.
    -r | --vreosversion     : VRE OS major version to use (e.g. 6, 7, or A [for all]) default = $VREOS_LABEL
    -U | --uefi-boot        : Force a UEFI boot stick to be made (default for HELiOS 7)
    -B | --bios-boot        : Force a BIOS boot stick to be made (default for HELiOS 6)

    --dont-eject            : Skips eject
    --skip-grub-install     : Skips the grub install onto the usb stick
    --ignore-unmount-errors : ignores unmount errors
    --grub-timeout          : Sets the grub timeout to support an auto boot.  Defaults to 30 seconds.  Only enabled for skip grub installs.
    --add-restore-hook      : Adds the restore hook for developer written restore scripts.
    --mrtf-non-vm           : Utilize MRTF mode, but assume a target/show-machine and not a VM. This option must occur after the MRTF flag.
    --robust                : Robust mode to enable Advanced Menu
    --logfile               : Force a specific location for the logfile
    --rias                  : Remote installer at site 
EOF
    exit 1
}

function resetDeviceMounts
{
    exitVal=0
    set +e
    for ydev in $DATADEV $EFIDEV $BIOSDEV; do
        [ -z "$ydev" -o ! -e "$ydev" ] && continue

        out=$(findmnt $ydev)
        exitVal=$?
        if [ "x$exitVal" == "x0" ]; then
            $SAFE_UNMOUNT_DEVICE $ydev || $IGNORE_UNMOUNT_ERRORS
            exitVal=$?
        else
            exitVal=0
        fi
    done
    return $exitVal
}

function resetAllMounts
{
    set +e
    for ydir in $TMPMOUNT $EFIMNT $DATAMNT $USBISO_MNT; do
        [ -z "$ydir" -o  ! -e "$ydir" ] && continue
        
        out=$(mountpoint -q -d $ydir)
        exitVal=$?
        if [ "x$exitVal" == "x0" ]; then
            $SAFE_UNMOUNT_FOLDER $ydir || $IGNORE_UNMOUNT_ERRORS
            exitVal=$?
        else
            exitVal=0
        fi
    done
    
    resetDeviceMounts
    if [ "x$exitVal" == "x0" ]; then
        exitVal=$?
    fi
    
    return $exitVal
}

function cleanUp 
{
    echo "Cleanup in progress, please wait...."
    sync
    resetAllMounts
    exitVal=$?
    
    if [ ! -z $FLOOPBACK ]; then
        losetup -D $DEVTOUSE
        exitVal=$?
        losetup -D $LOOPBACK
        [ "x$exitVal" == "x0" ] && exitVal=$?
        
        mv $LOOPBACK $FLOOPBACK
        [ "x$exitVal" == "x0" ] && exitVal=$?
    fi
    
    rm -fR ${TMPSTORE}
    [ "x$exitVal" == "x0" ] && exitVal=$?
    [ "X${MYLOOPLOCK}" == "X1" ] && rm -f $LOOPLOCK

    # This line is important to keep, as it is used for monitoring the state
    # of the seabee execution. Initially for use by remote-install.
    echo "__SEABEE-END__"

    return $exitVal
}

function endOfItAll 
{
    exitcode=$?
    set +e
    
    # Reset the trap
    trap - 0 1 2 3 15
    
    cleanUp
    [ "x$exitcode" == "x0" ] && exitcode=$?

    if [ "x$exitcode" != "x0" ]; then
        echo "ERROR: An error occurred while processing, please review the log for details"
        echo "LOGFILE: $LOGFILE"
    else
        set +e
        sync; sync;
        if [ $DONT_EJECT -ne 1 ]
        then
            echo "Safely eject/powering off $DEVTOUSE"
            $SAFE_REMOVE $DEVTOUSE
        fi
        echo "SUCCESS"
        set -e
    fi
    exit $exitcode
}

function validateUSBDevice
{
    # Turn error failures off, as usb* may not exist and generates an error within ls
    set +e
    usbcnt=$(ls /dev/disk/by-id/usb* | xargs -iFile readlink -f File | grep -c "${USBDEV}" ) 
    set -e
    
    if [ "x$usbcnt" == "x0" -a "x${MRTF}" != "x${MRTFENABLE}" ]; then
        echo "${USBDEV} was not found to be a USB device."
        echo "Please verify you have selected the correct device ${USBDEV}."

        # read sends the prompt to stderr so need to workaround the trace file
        set +exv
        read -p "Are you sure you wish to proceed with $USBDEV [y/n]: " usragree 2>&1
        set -exv
        
        if [ "x$usragree" != "xy" ]; then
            exit 1
        fi
    fi
    return 0
}

function partitionDevice
{
    devtopart=$1
    [ -z $devtopart ] && exit
    
    PARTS=0
    [ -e /dev/disk/by-partlabel/$BIOS_LABEL ] && PARTS=$(($PARTS + 1))
    [ -e /dev/disk/by-partlabel/$EFI_LABEL ] && PARTS=$(($PARTS + 1))
    [ -e /dev/disk/by-partlabel/$DATA_LABEL ] && PARTS=$(($PARTS + 1))

    if [ "x$PARTS" == "x3" ]; then
        [ "x"$(readlink -f /dev/disk/by-partlabel/$BIOS_LABEL) = "x${BIOSDEV}" ] && PARTS=$(($PARTS + 1))
        [ "x"$(readlink -f /dev/disk/by-partlabel/$EFI_LABEL) = "x${EFIDEV}" ] && PARTS=$(($PARTS + 1))
        [ "x"$(readlink -f /dev/disk/by-partlabel/$DATA_LABEL) = "x${DATADEV}" ] && PARTS=$(($PARTS + 1))
    fi
    
    [ "x$FORCE" == "x1" ] && PARTS=0

    if [ "x$PARTS" != "x6" ]; then
        # Wipe boot loader, MBR, and first copy of GPT
        dd if=/dev/zero of=$devtopart bs=512 count=64
        # Wipe out all copies of the GPT, not just the first
        sgdisk -oZ $devtopart 1>&2 || true

        if [ "x$BOOTMODE" == "xB" ]; then
            parted -- $devtopart mklabel msdos 1>&2
            parted -- $devtopart mkpart primary fat16 1M   51M 1>&2
            parted -- $devtopart mkpart primary fat16 51M  101M 1>&2
            parted -- $devtopart mkpart primary ext4  101M -1 1>&2
            parted -- $devtopart print 1>&2
        else
            sgdisk -n 1:0:+1MB -c 1:"$BIOS_LABEL" -t 1:ef02 $devtopart 1>&2
            sgdisk -n 2:0:+50MB -c 2:"$EFI_LABEL" -t 2:ef00 $devtopart 1>&2
            sgdisk -n 3:0:0 -c 3:"$DATA_LABEL" -t 3:ef00 $devtopart 1>&2
            sgdisk -A 3:set:2 $devtopart 1>&2
            sgdisk -p $devtopart 1>&2
            sgdisk -v $devtopart
        fi
    fi
}

function formatDevice
{
    set +e
    type1=$(file -sL $DATADEV | grep -ic ext4)
    type2=$(file -sL $EFIDEV | grep -ic mkfs.fat)
    set -e

    # sleep required so device is not marked as busy
    sleep 30

    [ "x$type1" != "x1" -o "x1" == "x$FORCE" ] && mkfs.ext4 $DATADEV 1>&2 
    [ "x$type2" != "x1" -o "x1" == "x$FORCE" ] && mkfs.vfat $EFIDEV 1>&2

    fatlabel $EFIDEV $EFI_LABEL
    e2label $DATADEV $DATA_LABEL
    
    mkdir -p $EFIMNT $DATAMNT $TMPMOUNT 1>&2

    mount $EFIDEV $EFIMNT 1>&2 
    mount $DATADEV $DATAMNT 1>&2 

    if [ $SKIP_GRUB_INSTALL -ne 1 ]
    then 
        # We always install EFI grub - this means that the stick will
        # continue to work with the --mrtf option on a VM as well as
        # meaning that we can manually launch the UEFI installer from a
        # UEFI shell even if we have a BIOS stick.  This manual launch
        # works at least on the HP Z420.
        $GRUBINSTALL --target=x86_64-efi --efi-directory=$EFIMNT --boot-directory=$DATAMNT/boot --removable --recheck
    fi
}

function initializeUSB
{
    DEVTOUSE=${USBDEV}
    BIOSDEV=$DEVTOUSE"1"
    EFIDEV=$DEVTOUSE"2"
    DATADEV=$DEVTOUSE"3"
    
    resetDeviceMounts
    partitionDevice $USBDEV
    formatDevice
}

function initializeLoopback
{
    [ -z $LOOPBACK ] && return 0
    
    truncate -s${LOOPBACK_SIZE}G ${LOOPBACK}
    partitionDevice $LOOPBACK

    DEVTOUSE=$(losetup --partscan --find --show $LOOPBACK)
    BIOSDEV=$DEVTOUSE"p1"
    EFIDEV=$DEVTOUSE"p2"
    DATADEV=$DEVTOUSE"p3"
    
    formatDevice
}

function appendGrub
{
    isoname=$1
    osfolder=$2
    isofile=$3
    osversion=$4
    vreiso=$5
    robust=$6

    mrtf="no"
    gehc_vm="gehc.vm=no"
    gehc_test="gehc.test=1"
    gehc_integrity="gehc.integrity=$INTEGRITY_CHECKS"
    gehc_rias="gehc.rias=0"
    
    [ "x$MRTF" == "x${MRTFENABLE}" ] && mrtf="yes"
    [ "x$VM" == "x1" ] && gehc_vm="gehc.vm=yes" 
    [ "x$ISPRODUCTION" == "x1" ] && gehc_test=""
    [ "x$RIAS" == "x1" ] && gehc_rias="gehc.rias=1"

    if [ "x$MODE" == "x?" ]; then
        MODE="A"
        [ "x$VBOX" == "x1" ] && MODE="U"
        [ "x$VM" == "x1" ] && MODE="U"
    fi

    osbuild=$BUILD_NAME"_"$isoname

    # For help in troubleshooting and debugging the detailed menu is helpful
    promptlbl="[apps]"$BUILD_NAME"[host]"${isoname%.*}"[vre]"${vreiso%.*}
    [ "x$robust" == "x1" ] && promptlbl=$promptlbl"[robust]"
    
    # For production mode, Service would like only the BUILD_NAME to be displayed
    [ "x$ISPRODUCTION" == "x1" ] && promptlbl=$BUILD_NAME
    [ "x$ISPRODUCTION" == "x1" -a "x$robust" == "x1" ] && promptlbl=$promptlbl":ROBUST"

    biosdevname="biosdevname=0"
    if [ "x$osversion" == "x7" ]; then
        biosdevname=""
    fi

cat >> $GRUB_FILE << EOA

menuentry '$promptlbl' {
    load_video
    set gfxpayload=auto

#   insmod search_fs_label
    search --no-floppy --set=isopart --label $DATA_LABEL

    loopback loop (\$isopart)$isofile
    linuxefi (loop)/isolinux/vmlinuz debug ks=hd:LABEL=$DATA_LABEL:/SEAMARK/GEHCMR-MR$osversion.cfg stage2=hd:LABEL=$DATA_LABEL:$osfolder repo=hd:LABEL=$DATA_LABEL:$osfolder modprobe.blacklist=nouveau $biosdevname mrtf=$mrtf osbuild=$osbuild vreiso=$vreiso $gehc_vm $gehc_test $gehc_integrity fips=1 installmode=$MODE mrexit=$robust $gehc_rias
    
    initrdefi (loop)/isolinux/initrd.img
}

EOA

    return 0
}

function createGrub
{
    GRUB_FILE=$DATAMNT/boot/grub2/grub.cfg
    mkdir -p $DATAMNT/boot/grub2
    cat > $GRUB_FILE << EOA

function load_video {
  if [ x\$feature_all_video_module = xy ]; then
    insmod all_video
  else
    insmod efi_gop
    insmod efi_uga
    insmod ieee1275_fb
    insmod vbe
    insmod vga
    insmod video_bochs
    insmod video_cirrus
  fi
}

EOA

    if [ $SKIP_GRUB_INSTALL -ne 0 ]
    then
        cat >> $GRUB_FILE << EOA
set timeout=$GRUB_TIMEOUT
EOA
    fi

    [ "x${MRTF}" == "x${MRTFENABLE}" ] || [[ "x${RIAS}" == "x1" && "x${MODE}" == "xU" ]] && echo "set timeout=30" >> $GRUB_FILE
    echo "" >> $GRUB_FILE
    
    robustval="0"
    for rmode in ${ALLOWROBUST[@]}; do
        for hl in ${OSVERSIONLST[@]}; do
            [ "x$hl" != "x$OS_LABEL" -a "x$OS_LABEL" != "xA" ] && continue
            
            for iso in $(find $DATAMNT/os/HELiOS$hl -maxdepth 1 -name '*.iso' -type f ); do 
                isoname=$(basename $iso)
                osfolder=/os/$(basename `dirname $iso`)
                isofile="$osfolder/$isoname"
                for vre in ${OSVERSIONLST[@]}; do
                    [ "x$vre" != "x$VREOS_LABEL" -a "x$VREOS_LABEL" != "xA" ] && continue

                    for viso in $(find $DATAMNT/SEAMARK/iso/VREOS$vre -maxdepth 1 -name '*.iso' -type f ); do 
                        vreiso=$(basename $viso)
                        appendGrub $isoname $osfolder $isofile $hl $vreiso $robustval
                    done

                done

                if [ "x$VREOS_LABEL" == "xno" ] ; then
                    appendGrub $isoname $osfolder $isofile $hl $VREOS_LABEL $robustval
                else
                    continue
                fi
            done
        done

        [ "x$ISPRODUCTION" == "x1" ] && break
        [ "x$rmode" == "xE" ] && continue
        echo "submenu 'Advanced Modes' {" >> $GRUB_FILE
        robustval="1"
    done
    
    [ "x${robustval}" == "x1" ] && echo "}" >> $GRUB_FILE

    return 0
}

function getHELiOSContent
{
    for MACH_OS in "HELiOS" "VREOS"; do
        # If asked to skip VRE by commandline then skip copying the content. This is 
        # mostly for development effort to quickly do OS kickstart testing
        [ "x$VREOS_LABEL" == "xno" -a $MACH_OS == "VREOS" ] && continue
    
        DESTDIR=''
        REQ_VER=''
        if [ "x$MACH_OS" == "xHELiOS" ]; then
            DESTDIR="$DATAMNT/os"
            REQ_VER=$OS_LABEL
        else
            DESTDIR="$DATAMNT/SEAMARK/iso"
            REQ_VER=$VREOS_LABEL
        fi
        mkdir -p $DESTDIR

        for x in ${OSVERSIONLST[@]}; do
            [ "x$x" != "x$REQ_VER" -a "x$REQ_VER" != "xA" ] && continue

            # TODO: hardlink host OS and VRE OS builds if they are the
            # same release number to save USB stick space
            echo "Copying ${MACH_OS} $x content"
            SRCDIR="$OS_CONTENT/${MACH_OS}$x"
            $SYNC_WITH_PROG $SRCDIR $DESTDIR

            # HELiOS 6 on the host OS needs access to the install.img file
            # to correctly load Anaconda from the USB stick - VRE OS does
            # not need this
            [ "x$x" == "x6" -a "x$MACH_OS" == "xHELiOS" ] && extractHELiOS6Boot
            echo "${MACH_OS} $x content copied"
        done
    done
}

function getHELiOSKickstart
{
    echo "Copying HELiOS kickstart"
    
    for x in ${OSVERSIONLST[@]}; do
        [ "x$x" != "x$OS_LABEL" -a "x$OS_LABEL" != "xA" ] && continue
        rm -f $DATAMNT/SEAMARK/GEHCMR-MR$x.cfg
        cp "${USBISO_MNT}/inst/GEHCMR-MR$x.cfg" "$DATAMNT/SEAMARK/GEHCMR-MR$x.cfg"
        cat "${USBISO_MNT}/inst/GEHCMR-MR$x.cfg-platform" >> "$DATAMNT/SEAMARK/GEHCMR-MR$x.cfg"
        [ "x$ISPRODUCTION" == "x0" ] && cat "${USBISO_MNT}/inst/GEHCMR-MR$x.cfg.tst-patch" >> "$DATAMNT/SEAMARK/GEHCMR-MR$x.cfg"
    done
        
    echo "HELiOS kickstart copied"
    echo ""
}

function getMRAppContent
{
    [ -z $APPS_LOC ] && return 0

    echo "Copying MR Application content...."
    mkdir -p $DATAMNT/SEAMARK/iso
    for x in ${DISCLIST[@]}; do

        # If production image, then don't copy the MrTest content
        [ "$x" == "MrTest" -a "x$ISPRODUCTION" == "x1" ] && continue

        file=$APPS_LOC/$BUILD_NAME/iso/$x
        if [ -d $file ]; then
            echo "Copying $x"
            $SYNC_WITH_PROG $file $DATAMNT/SEAMARK/iso
        else
            echo "Skipping $x"
        fi
    done

    if [ ! -z "${SAVEINFOFOLDER}" -a -d ${SAVEINFOFOLDER} ]; then
        echo "Copying saveINFO"
        mkdir -p $DATAMNT/SEAMARK/saveINFO
        $SYNC_WITH_PROG ${SAVEINFOFOLDER}/* $DATAMNT/SEAMARK/saveINFO
    fi

    echo "MR Application content copied"
}

function copyMrUSBOSInstContentToRepo
{
    OSVER=$1
    DESTDIR=$2

    mkdir -p $DESTDIR
    echo "Syncing repo-mr"
    $SYNC_WITH_PROG $USBISO_MNT/inst/HELiOS${OSVER}/repo/mr $DESTDIR
    echo "Syncing repo-other"
    $SYNC_WITH_PROG $USBISO_MNT/inst/HELiOS${OSVER}/repo/other $DESTDIR

    # If this is for testing, include some additional packages to help debugging
    if [ "x$ISPRODUCTION" == "x0" ] ; then
        echo "Syncing repo-mrtest"
        $SYNC_WITH_PROG $USBISO_MNT/inst/HELiOS${OSVER}/repo/mrtest $DESTDIR
    fi
    
    echo "MR USB Installer Repo content copied"
}

function copyMrUSBSEAMARKInstContentToRepo
{
    DESTDIR=$1

    mkdir -p $DESTDIR
    $SYNC_WITH_PROG $USBISO_MNT/inst/SEAMARK/repo/* $DESTDIR
}

function createRepoAndGroup
{
    GRPNAME=$1
    DESTDIR=$2
    # Important comps.xml must go in the folder with the RPMs, otherwise the createrepo command will
    # pass but the group will not be found and work.
    COMPFILE="${DESTDIR}/comps.xml"

    # Create Basic Repo
    createrepo -q --database --simple-md-filenames -o ${DESTDIR} ${DESTDIR}

    # Create Repo Package file, base stuff
    cat > ${COMPFILE} << EOA
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE comps PUBLIC "-//Red Hat, Inc.//DTD Comps info//EN" "comps.dtd">
<comps>
  <group>
    <id>$GRPNAME</id>
    <name>$GRPNAME</name>
    <default>true</default>
    <description>Auto generated package $GRPNAME</description>
    <uservisible>true</uservisible>
    <packagelist>
EOA

    pkglist=""
   
    # Following code can also iterate over RPMs in sub-folders of the repo if any, ie os/repo/mr, os/repo/other ...
    find $DESTDIR -type f -iname "*.rpm" -print0 | while IFS= read -r -d $'\0' rpm_pkg; do
        pkg=$(rpm -qip $rpm_pkg | grep Name  |awk -F: '{print $2}' | sed -e 's/^[[:space:]]*//')
        echo "<packagereq type=\"mandatory\">${pkg}</packagereq>" >> $COMPFILE  
    done

    cat >> ${COMPFILE} << EOA
    </packagelist>
  </group>
  <category>
    <id>GEHMR</id>
    <display_order>1</display_order>
    <name>GEHCMR Addons</name>
    <description>GEHC MR Packages and Groups</description>
    <grouplist>
      <groupid>${GRPNAME}</groupid>
    </grouplist>
  </category>
</comps>     
EOA

    # Createrepo -q --database --simple-md-filenames -o <dest> --groupfile <groupfile> <directory>
    createrepo -q --database --simple-md-filenames -o ${DESTDIR} --groupfile ${COMPFILE} ${DESTDIR}

}

function createRepoPackaging
{
    SMARKREPO="$DATAMNT/SEAMARK/repo"
    OSMARKBASE="$DATAMNT/os"

    copyMrUSBSEAMARKInstContentToRepo $SMARKREPO
    
    # Setup each OS REPO
    for x in ${OSVERSIONLST[*]}; do
        [ "x$x" != "x$OS_LABEL" -a "x$OS_LABEL" != "xA" ] && continue
        SRCDIR="$OSMARKBASE/HELiOS$x/repo"
        [ ! -d "${SRCDIR}" ] && mkdir -p "${SRCDIR}"
        copyMrUSBOSInstContentToRepo $x $SRCDIR
        createRepoAndGroup "mrhelios$x" ${SRCDIR}
    done

    # Setup repo for the SEAMARK folder
    if [ -d "${SMARKREPO}" ]; then
        createRepoAndGroup "wxgroup" ${SMARKREPO}
    fi
}

# Deploy the AIDE config to the USB's data partition and init the AIDE database.
function generateFileChecksumList
{
    SRC_AIDE_CONF="${USBISO_MNT}/inst/aide.conf"
    TGT_AIDE_DIR=$DATAMNT/aide
    TGT_AIDE_CONF=$TGT_AIDE_DIR/aide.conf
    mkdir -p $TGT_AIDE_DIR
    chmod 755 $TGT_AIDE_DIR
    sed "s#TMP_SEABEE_DIR_PLACEHOLDER#$TMPSTORE#" $SRC_AIDE_CONF > $TGT_AIDE_CONF
    chmod 444 $TGT_AIDE_CONF
    if [  "x${INTEGRITY_CHECKS}" == "x1" ]; then
        aide --config=$TGT_AIDE_CONF --init
    fi
}

function __main__
{
    TEMP=`getopt -o b:h:i:o:a:fl:s:d:t:zS:Vr:PUB --long buildname:,apps:,integrity:,osversion:,oslocation:,force,loopback:,loopsize:,usbdev:,mrtf:,vbox,Sinfo:,VM,vreosversion:,production,uefi-boot,bios-boot,unattended,attended,logdir:,dont-eject,skip-grub-install,ignore-unmount-errors,grub-timeout,add-restore-hook,robust,mrtf-non-vm,rias,logfile: -- "$@"`
    if [ $? != 0 ] ; then echo "Terminating..." >&2 ; exit 1 ; fi
    # Note the quotes around `$TEMP': they are essential!
    eval set -- "$TEMP"

    while true; do
        case "$1" in
            
            --robust )
                ALLOWROBUST=(0 E)
                shift
            ;;
            
            --dont-eject )
                DONT_EJECT=1
                shift
            ;;

            --skip-grub-install )
                SKIP_GRUB_INSTALL=1
                shift
            ;;
            
            --ignore-unmount-errors )
                IGNORE_UNMOUNT_ERRORS=1
                shift
            ;;

            --grub-timeout )
                GRUB_TIMEOUT=$2
                shift 2
            ;;
            
            --logdir )
                LOGFILE=$(readlink -f $2)"/$BNAME.$$.trace"
                shift 2
            ;;
            
            --logfile )
                LOGFILE=$2
                shift 2
            ;;

            -b | --buildname ) 
                BUILD_NAME=$2 ; shift 2
            ;;

            -a | --apps )
                APPS_LOC=${2%"${2##*[!/]}"} ; shift 2  # Remove any trailing slashes
            ;;

            -h | --osversion )
                OS_LABEL=$2 ; shift 2
            ;;

            -o | --oslocation )
                OS_CONTENT=${2%"${2##*[!/]}"} ; shift 2  # Remove any trailing slashes
            ;;

            -f | --force )
                FORCE=1 ; shift
            ;;

            -i | --integrity )
                if ! [[ "$2" =~ ^[01]$ ]]; then
                    echo "Invalid USB integrity check flag, exiting."
                    usage
                fi
                INTEGRITY_CHECKS=$2 ; shift 2
            ;;

            -l | --loopback )
                FLOOPBACK=$2 ; shift 2
            ;;
            
            -S | --Sinfo )
                SAVEINFOFOLDER=${2%"${2##*[!/]}"} ; shift 2  # Remove any trailing slashes
            ;;

            -s | --loopsize )
                LOOPBACK_SIZE=$2 ; shift 2
            ;;

            -d | --usbdev )
                USBDEV=${2%"${2##*[!/]}"} ; shift 2  # Remove any trailing slashes
            ;;
            
            --mrtf-non-vm )
                if [ "x$MRTF" != "x${MRTFENABLE}" ]; then
                    echo "Invalid MRTF usage, exiting."
                    usage
                fi
                
                VBOX=0
                VM=0
                shift
            ;;
            --rias )
                RIAS=1 ; shift
            ;;
            
            -t | --mrtf )
                if [ "x$2" != "x${MRTFENABLE}" ]; then
                    echo "Invalid MRTF id, exiting."
                    usage
                fi
                
                echo "###########################################"
                echo "### WARNING...WARNING....WARNING !!!!!! ###"
                echo "# YOU HAVE SELECTED THE MRTF OPTION"
                echo "# THIS OPTION IS DESTRUCTIVE TO THE SYSTEM"
                echo "# AND SHOULD NOT BE USED UNLESS IN A MRTF VM INSTALL"
                echo "#"
                echo "# CLICK ctrl-C within 10 seconds if you"
                echo "# are not a MRTF VM installer"
                echo "###########################################"
                secs=10
                while [ $secs -gt 0 ]; do
                   echo -ne "$secs\033[0K\r"
                   sleep 1
                   : $((secs--))
                done
                
                MRTF=$2
                VBOX=1
                VM=1
                shift 2
            ;;

            -z | --vbox )
                VBOX=1 ;VM=1 ; shift
            ;;
            -V | --VM )
                VM=1 ; shift
            ;;
            -r | --vreosversion )
                VREOS_LABEL=$2 ; shift 2
            ;;

            -P | --production )
                ISPRODUCTION=1 ; shift
            ;;

            -U | --uefi-boot )
                BOOTMODE='U'; shift;
            ;;

            -B | --bios-boot )
                BOOTMODE='B'; shift;
            ;;

            --attended )
                MODE='A' ; shift;
            ;;

            --unattended )
                MODE='U' ; shift;
            ;;

            -- )
                shift; break
            ;;
            
            *)
                echo $1
                usage
            ;;
        esac
    done

    # Do not allow integrity checking to be disable for production sticks.
    if [ $ISPRODUCTION -eq 1 -a $INTEGRITY_CHECKS -ne 1 ]; then
        echo "Cannot disable integrity checks for a production stick, exiting."
        usage
    fi

    exec 2>$LOGFILE && set -xve
    resetAllMounts

    BOOTMODE="U"
    
    USBISO_LOC="${APPS_LOC}/${BUILD_NAME}/iso/MrUSBInstaller/${BUILD_NAME}-MrUSBInstaller.iso"
    mkdir -p ${USBISO_MNT}
    mount ${USBISO_LOC} ${USBISO_MNT}
    
    if [ -z ${BUILD_NAME} ]; then
        usage
    elif [ -z ${OS_CONTENT} ]; then
        [ ! -f ${USBISO_LOC} ] && echo "Cannot find ${USBISO_LOC}" && exit 1
        OS_CONTENT="${USBISO_MNT}/inst"
    fi

    # Prevent accidents
    validateUSBDevice

    #+++++++++++++++++++++++++++++++++++++++++++++++++
    echo "Partitioning, please wait...."
    if [ ! -z ${USBDEV} ]; then
        initializeUSB
    else
        initializeLoopback
    fi
    echo "Completed Partitioning of $DEVTOUSE"
    echo ""
    #-------------------------------------------------

    #+++++++++++++++++++++++++++++++++++++++++++++++++
    echo "Copying content, please wait..."
    getHELiOSContent
    getHELiOSKickstart
    getMRAppContent
    createGrub
    echo "Content setup complete."
    echo ""
    #-------------------------------------------------

    #+++++++++++++++++++++++++++++++++++++++++++++++++
    echo "Generating repos..."
    createRepoPackaging
    echo "Repos generated."
    echo ""
    #-------------------------------------------------
    
    #+++++++++++++++++++++++++++++++++++++++++++++++++
    echo "Generating integrity checks, please wait..."
    generateFileChecksumList
    echo "Integrity check generation complete."
    echo ""
    #-------------------------------------------------

    # Copy the necessary files overtop of the existing OS grub so that the isntaller
    # is started without having to go into the BIOS settings. This is not 100% product,
    # but utilizes the same basic principles and config files so it is close enough to ensure
    # adequate runtime testing of the installer.
    if [ "x$MRTF" == "x${MRTFENABLE}" ]; then
        cp $GRUB_FILE /boot/efi/EFI/redhat/grub.cfg
        cp -R $DATAMNT/boot/grub2/x86_64-efi /boot/efi/EFI/redhat
    fi
    return 0
}

trap endOfItAll 0 1 2 3 15
__main__ $@
exit $?

```

# createUSB-3device 
:::tip
🎽 createUSB-3device 
source code 2022-06-23
:::

``` shell
#!/bin/bash
#
# Copyright 2021 General Electric Company.  All rights reserved.
# Wed Jul 14 06:29:05 EDT 2021
#


start_time=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`

# check how many other block devices
NUM_DEVICE=`ls /dev/sd? | wc -l`
NUM_DEVICE=`lsblk | grep ^sd. | wc -l`
NUM_OTHER_DEVICE=$((${NUM_DEVICE} - 2))
DONT_EJECT=0

while getopts ':b:r:d:p:e:h' opt
do
        case $opt in
        b)
                build=$OPTARG
                ;;cd
        r)
                restore=$OPTARG
                ;;
        d)
                device=$OPTARG
                ;;
        e)
                DONT_EJECT=1
                ;;
        p)
                product_mode=$OPTARG
                ;;
        h)
                echo "usage: Ex: createUSB -b MR27.0_EA_1904.a -r /tmp/restoreinfo -d /dev/sdb"
                echo "    -b build name"
                echo "    -r restore info path"
                echo "    -d usb device name"
                echo "    -e don't eject usb devices"
                echo "    -p product mode if -p 1; otherwise develop mode"
                exit 0
                ;;
        esac
done

if [ -z "$build" ]; then
        echo "please input build version"
        exit 1
fi
FILE_TIMESTAPM=`date +%Y_%m_%d_%H:%M`
BUILD_HISTORY_DIR="/home/sdc/creat_3USB_device/${build}_${FILE_TIMESTAPM}_trace"
SHA_VALUE_FILE="${BUILD_HISTORY_DIR}/sha_value.txt"
CREATE_LOG_FILE="${BUILD_HISTORY_DIR}/create_usb.log"

if [ ! -d "$BUILD_HISTORY_DIR" ]
then
        mkdir "$BUILD_HISTORY_DIR"
fi


function creat_installer_dir{
        echo "create installer directory ......"        echo "create installer directory ......" >> "$CREATE_LOG_FILE"
        mntpoint=/tmp/usbiso
        creatorhome=/tmp/usbcreator

        if [ ! -d $mntpoint ]
        then
                mkdir $mntpoint
        fi

        if [ ! -d $creatorhome ]
        then
                mkdir $creatorhome
        fi
        # prepare clean work enviroment
        umount $mntpoint
        echo "11111111$creatorhome/*"
        rm -rf $creatorhome/*

        installer=/b/pvr2/$build/iso/MrUSBInstaller/$build-MrUSBInstaller.iso
        if [ ! -f "$installer" ]
        then
                echo "can not find installer"
                echo "can not find installer" >> "$CREATE_LOG_FILE"
                exit 1
        fi

        mount $installer $mntpoint
        cp -a $mntpoint/* $creatorhome
        umount $mntpoint

        seabee=$creatorhome/inst/seabee
        if [ ! -f "$seabee" ]
        then
                echo "can not find seabee"
                echo "can not find seabee" >> "$CREATE_LOG_FILE"
				umount $mntpoint
                exit 1
        fi
}

function echo_time
{
        echo -e "\033[5;32m*************************************************************************************\033[0m"
        echo -e "\033[5;32m*                                                                                   *\033[0m"
        echo -e "\033[5;32m*\033[0m    Master execution duration: $master_duration     \033[5;32m*\033[0m"
        echo -e "\033[5;32m*\033[0m    Sub execution duration: $sub_duration  \033[5;32m*\033[0m"
        echo -e "\033[5;32m*                                                                                   *\033[0m"
        echo -e "\033[5;32m*************************************************************************************\033[0m"
        echo "Master execution duration: $master_duration"  >> "$CREATE_LOG_FILE"
        echo "Sub execution duration: $sub_duration" >> "$CREATE_LOG_FILE"
}


function main
{
        # accept an argument : usb device node
        cmd="$seabee --usbdev $1 --oslocation $creatorhome/inst --buildname $build --force --dont-eject"
        echo $cmd
        if [ -n "$restore" ]
        then
                cmd="$cmd -S $restore"
        fi

        if [ "$product_mode"x = "1"x ]
        then
                cmd="$cmd -P"
        fi
        $cmd
        finish_time_master=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
        master_duration=$((($(date +%s -d "$finish_time_master") - $(date +%s -d "$start_time"))/60))mins
        umount $mntpoint
}


if [ $NUM_OTHER_DEVICE = 1 ]
then
        echo -e "\033[31m You only insert one USB devices, please use the script </root/createUSB> \033[0m"
        exit 1
fi

if [ "$NUM_OTHER_DEVICE" = 2 ]
then
        echo -en "\033[33m You only insert two USB devices, only generate two system boot USB; are you sure? [y/n] \033[0m"
        read FLAG
        if [ "$FLAG" = y -o "$FLAG" = Y ]
        then
                device_master=`ls -l /dev/sd? | tail -2 | head -1 | awk 'END {print $NF}'`
                if [ -n "$device_master" ]
                then
                        creat_installer_dir
                        main $device_master
                else
                        echo -e "\033[31m Dont detect a valid device \033[0m"
                        echo -e "\033[31m Dont detect a valid device \033[0m" >>        "$CREATE_LOG_FILE"
                        exit 1
                fi

                device_sub=`ls -l /dev/sd? | tail -1 | awk 'END {print $NF}'`
                if [ -n $device_sub ]
                then
                        echo "generating sub-disk by dd command ......"
                        echo "generating sub-disk by dd command ......" >> "$CREATE_LOG_FILE"
                        `dd bs=1M if=$device_master of=$device_sub`
                        sync
#:<<!
                        sha_time1=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
                        sha_value_master=`sha256sum $device_master | awk '{print $1}'`
                        sha_value_sub=`sha256sum $device_sub | awk '{print $1}'`
                        sha_time2=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
                        sha_duration=$(($(date +%s -d "$sha_time2") - $(date +%s -d "$sha_time1")))seds

                        if [ $sha_value_master = $sha_value_sub ]
                        then
                                echo -e "\033[32m sub_USB sha256 value equal master sha256 value,\n value: $sha_value_sub\033[0m"
                                echo -e "\033[32m sub_USB sha256 value equal master sha256 value,\n value: $sha_value_sub\033[0m" >> "$CREATE_LOG_FILE"
                                 echo "${build}: ${sha_value_sub}" > "$SHA_VALUE_FILE"
                        else
                                echo -e "\033[31m Note sub_USB sha256 value not equal master sha256 value, data may be tampered\033[0m"
                                echo -e "\033[31m Note sub_USB sha256 value not equal master sha256 value, data may be tampered\033[0m" >> "$CREATE_LOG_FILE"
                                echo $sha_value_master
                                echo $sha_value_sub
                                echo $sha_value_master >> "$CREATE_LOG_FILE"
                                echo $sha_value_sub >> "$CREATE_LOG_FILE"
                        fi
#!
                else
                        echo -e "\033[31m Dont detect a valid device \033[0m"
                        echo -e "\033[31m Dont detect a valid device \033[0m" >> "$CREATE_LOG_FILE"
                        exit 1
                fi

                finish_time_sub=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
                sub_duration=$((($(date +%s -d "$finish_time_sub") - $(date +%s -d "$finish_time_master"))/60))mins
                eject $device_master
                eject $device_sub

        fi

fi


if [ $NUM_OTHER_DEVICE = 3 ]
then
        device_master=`ls -l /dev/sd? | tail -3 | head -1 | awk 'END {print $NF}'`
        if [ -n "$device_master" ]
        then
                creat_installer_dir
                main $device_master
                echo "create master device done~"
                echo "create master device done~" >> "$CREATE_LOG_FILE"
                # if [ "$?" == 0 ]
                # then
                #       echo "create master device done~"
                #       echo "create master device done~" > "$CREATE_LOG_FILE"
                # else
                #       echo  -e "\033[31m Create master USB device failed  \033[0m"
                #       echo  -e "\033[31m Create master USB device failed  \033[0m" > "$CREATE_LOG_FILE"
                #       exit 1
                # fi
        else
                echo -e "\033[31m Dont detect a valid device \033[0m"
                echo -e "\033[31m Dont detect a valid device \033[0m"  >> "$CREATE_LOG_FILE"
                exit 1
        fi
        device_sub1=`ls -l /dev/sd? | tail -2 | head -1 | awk 'END {print $NF}'`
        device_sub2=`ls -l /dev/sd? | tail -1 | awk 'END {print $NF}'`

        echo "generating sub-disk by dd command ......"
        echo "generating sub-disk by dd command ......"  >> "$CREATE_LOG_FILE"
        dd bs=1M if="$device_master" of="$device_sub1" &
        dd bs=1M if="$device_master" of="$device_sub2"
        sleep 2
		
        dd_pid=`ps aux | grep "if=$device_master" | grep -v grep | awk '{print $2}' | head -1`
        while [ -n "$dd_pid" ]
        do
                echo "wait 15 seds for dd command complet ..."
                echo "wait 15 seds for dd command complet ..."  >> "$CREATE_LOG_FILE"
                sleep 15
                dd_pid=`ps aux | grep "if=$device_master" | grep -v grep | awk '{print $2}' | head -1`
        done

        echo "dd command done ~"
        echo "dd command done ~" >> "$CREATE_LOG_FILE"
        sync
        sync
#:<<!
        sha_time1=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
        sha_value_master=`sha256sum $device_master | awk '{print $1}'`
        sha_value_sub1=`sha256sum $device_sub1 | awk '{print $1}'`
        sha_value_sub2=`sha256sum $device_sub2 | awk '{print $1}'`
        sha_time2=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
        sha_duration=$(($(date +%s -d "$sha_time2") - $(date +%s -d "$sha_time1")))seds

        if [ "$sha_value_master" = "$sha_value_sub1" -a "$sha_value_master" = "$sha_value_sub2" ]
        then
                echo -e "\033[32m sub_USB sha256 value equal master_USB sha256 value,\n value: $sha_value_sub1\033[0m"
                echo -e "\033[32m sub_USB sha256 value equal master_USB sha256 value,\n value: $sha_value_sub1\033[0m" >> "$CREATE_LOG_FILE"
                echo "${build}: ${sha_value_sub1}" > "$SHA_VALUE_FILE"
        else
                echo -e "\033[31m Note sub_USB sha256 value not equal master sha256 value, data may be tampered\033[0m"
                echo -e "\033[31m Note sub_USB sha256 value not equal master sha256 value, data may be tampered\033[0m" >>"$CREATE_LOG_FILE"
                echo $sha_value_master
                echo $sha_value_sub1
                echo $sha_value_sub2
                echo $sha_value_master >> "$CREATE_LOG_FILE"
                echo $sha_value_sub1 >> "$CREATE_LOG_FILE"
                echo $sha_value_sub2 >> "$CREATE_LOG_FILE"
        fi
#!

        finish_time_sub=`date --date='0 days ago' "+%Y-%m-%d %H:%M:%S"`
        sub_duration=$((($(date +%s -d "$finish_time_sub") - $(date +%s -d "$finish_time_master"))/60))mins
        eject $device_master
        eject $device_sub1
        eject $device_sub2

fi

echo_time
```


# chapter12
:::tip
🎽 chapter12
source code 2022-06-23
:::

```shell
#/bin/bash

# This script: show how many days left for your birthday
# Version 1.0.0
# Author: snail  Reach out sso: 503247565
# First release  Jun 14 20:00 CST 2022

function leftDays(){
        birthday=$( date +%Y )$1
        today=$( date +%Y%m%d )
        birthday_seds=$( date --date="${birthday}" +%s )
        today_seds=$( date --date="${today}" +%s )
        # before birthday
        if [ "$today_seds" -lt "$birthday_seds" ]
        then
                left_days=$(( ($birthday_seds - $today_seds) / 60 / 60 / 24 ))
                echo "after $left_days days, you will have birthday"
        elif [  "$today_seds" -eq "$birthday_seds" ]
        then
                echo "today is your birthday"
        else
                # assume every year is 365 days
                left_days=$(( (365*24*60*60 + $birthday_seds - $today_seds) / 60 / 60 / 24 ))
                echo "after $left_days days, you will have birthday"
        fi
}

read -p "please input your birthday xxxx 0809: " input_day
leftDays $input_day


function systemUser(){
        lineNum=1
        for i in `cat /etc/passwd | cut -d ':' -f1`
        do
                echo "The $lineNum account is \"$i\""
                lineNum=$(( $lineNum + 1))
        done
}
systemUser

```