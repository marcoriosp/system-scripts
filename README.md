# system-scripts

Script from https://github.com/SergKolo/sergrep


The basic steps of how to create a swap file are described in Arch Wiki article on swap. I took the liberty of condensing all those steps into a script。 

### Basic usage:

Usage is very simple:

   ```sh
   sudo ./addswap.sh INTEGER LETTER
   ```

For adding 1 gigabyte you would do sudo ./addswap.sh 1 G. For adding 1 megabyte do sudo ./addswap.sh 1 M.

### Script
This script is also available on SergKolo personal GitHub repository (https://github.com/SergKolo/sergrep).

   ```sh
   #!/bin/bash
   set -e  # bail if anything goes wrong

   is_root(){
      if [ "$( id -u )" -ne 0  ] ; then
         return 1
      fi
      return 0
   }

   get_swap_amount(){
       # Obtain amount of swap in Gigabytes
       awk '/SwapTotal/{printf "%.2f",($2/1048576)}' /proc/meminfo
   }

   make_swap_file(){ 
      # This is the function that does the job of creatig swap file
      # and enabling it.  All files are timestamped
      printf "Current swap ammount: %f\n" "$(get_swap_amount)"
      printf "Working on creating swap file\n"
      DATE=$(date +%s) # append date of creation to filename
      filename="/swapfile.""$DATE" # File will be /swapfile.$DATE
      dd if=/dev/zero  of="$filename" bs=1"$2" count="$1"
      chmod 600 "$filename"
      mkswap "$filename" && 
      swapon "$filename" && 
      printf "\nCreated and turned on %s\n"  "$filename"
      printf "Current swap ammount: %f" "$(get_swap_amount)"
   }


   ask_to_enable_on_boot(){
      # Prompt user to enable this new swap file on boot. If user
      # enters y, the swap file will be added to /etc/fstab
      printf "Do you want to turn on this file at boot?[y/n]\n"
      read ANSWER
      case "$ANSWER" in
       [Yy]) printf "\n%s none swap defaults 0 0\n" "$filename"  >> /etc/fstab &&
          printf "\n %s added to /etc/fstab successfuly\n" "$filename"
          exit 0 ;;
       [Nn]) printf "Exiting\n" && exit 0 ;;
       *) printf "Wrong input: %s . Exiting. /etc/fstab not altered\n" "$ANSWER" && exit 1 ;;
      esac

   }

   bad_arguments(){
        # check if second argument is a character 
        case "$2" in 
            [A-Z]) return 1;;
            *) return 0;;
        esac

       # Check if first argument is a digit. 
       # https://stackoverflow.com/a/3951175/3701431
       case "$1" in
           ''|*[!0-9]*) return 0;;
           *) return 1 ;;
       esac 

   }

   main(){

       # Check if we're root and if args are OK. If everything is ok, do stuff

       if is_root 
       then
           if [ $# -ne 2   ] ||  bad_arguments "$@"
           then
               printf "%s\n" ">>> ERR: $0: bad or insufficient arguments" > /dev/stderr
               printf "%s\n" ">>> Usage: $0 INTEGER LETTER" > /dev/stderr
               exit 2
           fi
           make_swap_file "$@" && ask_to_enable_on_boot
       else
           printf ">>> ERR: $0 must run as root\n" > /dev/stderr
           exit 1
       fi
   }

   main "$@"
   ```

### Test run

   ```sh
   sudo ./addswap.sh 1 G

   # Response
   Current swap ammount: 4.000000
   Working on creating swap file
   1+0 records in
   1+0 records out
   1073741824 bytes (1.1 GB, 1.0 GiB) copied, 6.83322 s, 157 MB/s
   Setting up swapspace version 1, size = 1024 MiB (1073737728 bytes)
   no label, UUID=8bf1d78d-0a8a-478b-a783-38c5935c362f

   Created and turned on /swapfile.1498976162
   Current swap ammount: 5.000000Do you want to turn on this file at boot?[y/n] [Press Y]

    /swapfile.1498976162 added to /etc/fstab successfuly

   ```
